# Module Teardown: [Module Name / Component]

## 1. High-Level Overview
* **Core Responsibility:** [A 2-3 sentence summary of what this component actually does in the Trino engine]
* **Key Triggers:** [What causes this component to act? e.g., an HTTP request, a timer, a method call from an upstream operator]

## 2. Structural Architecture
* **Primary Source Files:** [List the top 3-5 crucial `.java` files]
* **Key Data Structures:** [What internal structures hold state? e.g., ConcurrentHashMap, specific queues, custom arrays]

### Class Diagram
```mermaid
classDiagram
    %% Claude: Insert Mermaid.js class diagram syntax here showing inheritance and composition.
```

## 3. Execution & Call Flow

### Sequence Diagram
```mermaid
sequenceDiagram
    %% Claude: Insert Mermaid.js sequence diagram syntax here mapping the primary execution path.
```

* **Step-by-step text breakdown:**
  1. [Step 1]
  2. [Step 2]
  3. [Step 3]

## 4. Concurrency & State Management
* **Threading Model:** [Does this run on a dedicated thread? Is it part of the driver loop? Does it block?]
* **State Machine:** [If applicable, what are the states this component transitions through? e.g., NEW -> RUNNING -> FINISHED]
* **Synchronization:** [Are there explicit locks, synchronized blocks, or volatile variables used here?]

## 5. Memory & Resource Profile
* **Allocation Pattern:** [Does this allocate large chunks of off-heap memory? Does it create many small objects?]
* **Memory Tracking:** [How does this component report its memory usage to the LocalMemoryContext or MemoryPool?]

## 6. Porting Considerations (Java -> Target Architecture)
* **Translation Blockers:** [Identify heavy reliance on Java-specific features like GC, JNI, or deep inheritance]
* **Recommended Abstractions:** [Suggest architectural patterns for the Rust rewrite, such as specific traits, Tokio async tasks, or memory allocator wrappers]
