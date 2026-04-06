# Meta Velox: Implementation Challenges & Architecture

Building Velox as a unified, high-performance C++ execution engine to serve fundamentally different host databases (Presto, Spark, etc.) revealed several critical friction points. Here are the primary challenges and architectural solutions detailed by the Velox team.

## 1. The Steep Learning Curve of Vectorized Code
One of the biggest hurdles was developer productivity and reliability. Writing highly optimized, vectorized columnar code is exceptionally difficult. Developers adding custom functions (UDFs) to Velox had to manually manage nullability bitmaps, handle different columnar encodings (like dictionary or run-length encoding), and pre-allocate memory buffers.

Because of this complexity, the Velox team noticed a disproportionate number of bugs, memory leaks, and performance regressions being introduced by contributors.
* **The Solution:** They built the **Simple Function API**. This allows developers to write standard, row-by-row C++ functions (ignoring columnar complexity entirely). Velox then automatically wraps, vectorizes, and optimizes these simple functions under the hood to operate efficiently on columnar data batches.

## 2. The Semantic Gap Between SQL Dialects
Velox was designed to replace the execution engines of both Presto and Apache Spark. However, these systems do not evaluate SQL in the exact same way. They have subtle but strict semantic differences known as "dialects." For example:
* **Error Handling:** How does the engine respond to a division by zero, or an invalid type cast (e.g., casting the string `"abc"` to an `INTEGER`)? One engine might throw a fatal error, while another might suppress the error and return a `NULL`.
* **Type Promotion:** How are decimal precision and scale calculated when multiplying two numbers?

* **The Solution:** The team had to build an incredibly flexible type system and a modular function registry. Instead of hardcoding behavior, Velox allows host engines to register "Spark-compatible" or "Presto-compatible" function packages at runtime, ensuring the core execution engine remains dialect-agnostic.

## 3. String Representation and Out-of-Order Writes
Velox relies heavily on the Apache Arrow columnar format, but the standard Arrow layout for Strings (a contiguous byte buffer with a separate offsets array) proved to be a major bottleneck for certain database operations.

In operations like building a hash table or sorting, the engine often needs to write strings out of order. Standard Arrow buffers require data to be written sequentially, which forces expensive data copying and buffering.
* **The Solution:** Velox deviated slightly from the Apache Arrow standard for strings. They implemented a custom 16-byte `StringView` structure. This structure holds a 4-byte inline prefix of the string (which speeds up string comparisons tremendously without following a pointer) and points to dynamically allocated, non-contiguous memory blocks, allowing for highly efficient out-of-order writes.

---

## 4. Deep Dive: Memory Management Across Language Boundaries

A common point of confusion is what Velox actually *is*. Velox itself is **not a worker daemon**; it is an embeddable C++ execution *library* (much like DataFusion is a Rust library).

When Meta built a standalone C++ worker that replaces the Java worker and communicates with the Java Coordinator via HTTP, they called that specific standalone worker **Prestissimo**. When running Prestissimo, the whole worker process is 100% C++, and the Java Coordinator just sends high-level HTTP limits (e.g., "Don't exceed 40GB for this query").

However, other massive systems—specifically **Apache Spark** (via the Intel Gluten project) and early versions of Presto—embed Velox directly **inside the Java process** using JNI (Java Native Interface). It is in these embedded use cases where the "Language Boundary" memory nightmare occurs.

### The "Invisible Memory" Problem
When you run a Java application like a Spark Executor, you give it a strict memory limit, say `-Xmx64G` (64 GB). The JVM Garbage Collector meticulously tracks every Java object inside that 64GB heap.

But when Java calls Velox via JNI to execute a SQL query, Velox is running in C++. When Velox needs to build a massive Hash Table for a join, it allocates memory directly from the operating system (using `mmap` or `malloc`). This is "off-heap" memory.

**The JVM has absolutely no visibility into C++ allocations.** If the JVM is using 60 GB of its heap, and Velox decides it needs 20 GB of C++ memory for a join, the total process now consumes 80 GB. If the container only has 64 GB of physical RAM, the Linux kernel's OOM (Out-of-Memory) Killer will step in and instantly crash the entire Spark Executor.

### The Solution: The JVM as the "Banker"
To prevent Velox from silently crashing the JVM process, the Velox team had to design their C++ `MemoryPool` and `MemoryAllocator` with cross-language hooks.

In an embedded system like Spark/Gluten, Velox is not allowed to just take memory from the OS. Instead:
1. Velox calculates it needs 1 GB for a batch of data.
2. Velox triggers a callback across the JNI boundary to the Java memory manager.
3. It asks Java: *"I need 1 GB. Do we have enough total process memory left?"*
4. The Java memory manager checks its global accounting tree. If yes, it grants permission, and Velox performs the C++ allocation. If no, the Java manager tells Velox to wait, or forces Velox to spill to disk.
