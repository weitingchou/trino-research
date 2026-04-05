# Phase 2: Trino Execution Model Overview

When a SQL query hits the Trino Coordinator, it doesn't just send the raw SQL to the workers. It compiles, optimizes, and shatters the query into a highly coordinated, distributed execution plan. This phase bridges the gap between the logical query and the physical hardware.

## 1. The Execution Hierarchy: From Query to Hardware

To understand Trino's execution, you have to understand its structural hierarchy. Every piece of work belongs to a parent container.

* **Query:** The grand orchestrator. This is the entire SQL statement submitted by the client, along with its configuration and session properties.
* **Stage:** The query is sliced horizontally into **Stages**. A Stage is a conceptual fragment of the query plan that can be executed independently.
    * *Crucial distinction:* A Stage does not run on a specific node. It is a distributed template. If your query involves joining two tables, scanning Table A is one Stage, scanning Table B is another, and joining them is a third.
* **Task:** This is where the conceptual becomes physical. A Task is the instantiation of a Stage on a *specific* Worker node. If a Stage is a blueprint, the Task is the actual construction crew on a specific site. Everything below this level lives inside a single worker process.

## 2. The Bridge: Splits and Scheduling

How does Trino decide *which* worker gets *which* Task, and what data they process?

* **Splits:** A Split is a specific chunk of the underlying data (e.g., a specific file in S3, a range of rows in a database, a partition in Iceberg). Trino doesn't assign workers to tables; it assigns workers to Splits.
* **Source Stages vs. Downstream Stages:**
    * **Source Stages:** These talk directly to the storage layer via Connectors. The Coordinator evaluates the table metadata, generates hundreds or thousands of Splits, and schedules Tasks on the workers to process those specific Splits.
    * **Downstream Stages:** These stages don't read from storage; they read intermediate data from other workers (e.g., an aggregation or a join stage).

## 3. The Nervous System: Data Movement (Exchanges)

Trino executes primarily in memory, with disk spilling as a fallback under memory pressure. This makes the execution model heavily reliant on network streams to move data between Stages.

* **Exchanges:** An Exchange is the mechanism that transfers data between nodes.
* **Upstream vs. Downstream:** As a Task processes its Splits, it pushes the resulting data into an output buffer. Tasks from the *next* Stage (downstream) continuously pull data from those buffers over the network.
* **Pipelined Execution:** This data movement happens continuously. Trino does not wait for an entire Stage to finish before starting the next one. As soon as the first Driver in a Source Stage produces a page of data, the Exchange mechanism streams it to the Downstream Stage, meaning multiple Stages execute concurrently.

## 4. Inside the Worker: Pipelines, Drivers, and Operators

Sections 1-3 describe the *distributed* structure. But the heart of Phase 2 is what happens *inside* a single worker after it receives a Task. This is where the actual computation happens.

### The Pipeline Taxonomy

When a Task is created on a worker, the plan fragment is compiled into one or more **Pipelines**. Each Pipeline is a chain of **Operator** factories (scan, filter, hash build, hash probe, output, etc.). The number and type of Pipelines depend on the plan — a simple scan-filter-project has one Pipeline, while a hash join has separate Pipelines for the build and probe sides.

Pipelines come in three flavors, each with a different **Driver spawning strategy**:

| Pipeline Type | Driver Count | Example |
|---|---|---|
| **Split-lifecycle** | One Driver per Split (dynamic, grows as Splits arrive) | Table scans — the Coordinator sends Splits incrementally |
| **Task-lifecycle** | Fixed count at Task start (default: `task_concurrency`, typically a power of two up to 32) | Hash-build side of joins, output buffers, exchange writes |
| **Remote-source** | Fixed count, Splits distributed round-robin to existing Drivers | Exchange inputs pulling data from upstream workers |

A **Driver** is the actual unit of work: a linear chain of instantiated Operators that processes Pages (batches of columnar data). Each Driver is single-threaded — only one thread works with its Operators at a time.

### The Operator Chain

Inside a Driver, Operators are arranged in a pipeline:

```
SourceOperator → FilterOperator → ProjectOperator → ... → SinkOperator
```

Each Operator implements a simple non-blocking state machine: `needsInput()`, `addInput(page)`, `getOutput()`, `isFinished()`, and `isBlocked()`. The Driver loop pulls pages from each operator and pushes them to the next, one pair at a time. When the terminal operator finishes, the Driver begins its shutdown sequence.

## 5. Cooperative Scheduling: The Engine Heart

Trino runs potentially thousands of Drivers on a fixed-size thread pool (default: `2 * CPU cores`). It achieves this through **cooperative multitasking** — Drivers voluntarily yield control so other Drivers get a turn.

### The Yield Loop

The core execution cycle works like this:

1. A **Runner thread** pulls the highest-priority Driver (wrapped as a `PrioritizedSplitRunner`) from the global scheduling queue.
2. The Runner calls `driver.processFor(1 second)`, which enters the cooperative loop:
   - Move Pages between adjacent Operators in the pipeline
   - If an Operator is **blocked** (waiting for network data, memory, or a hash table build), the Driver returns a `ListenableFuture` to the executor — "wake me when this completes"
   - If the **1-second quantum expires** (checked via a `DriverYieldSignal` that Operators cooperatively poll), the Driver yields control
   - If the Driver's pipeline is **finished**, it enters the destruction sequence
3. The Runner thread either re-enqueues the Driver for another quantum, parks it in a blocked set, or cleans it up.

**The key invariant: a Driver never blocks a thread.** It always returns control to the executor, either with "I'm done for now" (re-enqueue me) or "I'm waiting on something" (here's a future to watch). This is what allows a small thread pool to service thousands of concurrent Drivers.

### The Multilevel Feedback Queue

Not all Drivers are equal. Trino uses a **5-level priority queue** to ensure fairness across queries with very different lifetimes:

| Level | Accumulated Time | Typical Workload |
|---|---|---|
| 0 | 0 – 1 second | Short interactive queries |
| 1 | 1 – 10 seconds | Medium queries |
| 2 | 10 – 60 seconds | Complex joins/aggregations |
| 3 | 1 – 5 minutes | Heavy ETL-style queries |
| 4 | 5+ minutes | Long-running batch work |

The scheduler picks the level with the worst ratio of actual-to-target time, where targets decrease exponentially (factor of 2x per level). The result: **Level 0 gets 16x the CPU share of Level 4.** A freshly-arrived short query will get rapid service even when the cluster is saturated with long-running batch work.

## 6. The Task Lifecycle: Birth, Execution, and Death

A Task on a worker goes through a well-defined state machine:

```
RUNNING → FLUSHING → FINISHED
   │
   ├──→ CANCELING → CANCELED
   ├──→ ABORTING  → ABORTED
   └──→ FAILING   → FAILED
```

The happy path is straightforward: the Task starts in `RUNNING`, transitions to `FLUSHING` when all Drivers have completed (meaning data is still draining through the output buffer to downstream consumers), and finally reaches `FINISHED` when all downstream consumers have acknowledged receipt.

The unhappy paths use a **two-phase termination** protocol. When a failure or cancellation occurs, the Task enters an intermediate state (`FAILING`, `CANCELING`, or `ABORTING`) and waits for all live Drivers to shut down. Only when every Driver has been destroyed — releasing all memory back up the hierarchy (Operator → Driver → Pipeline → Task → Query) — does the Task transition to its terminal state. This ensures no resource leaks, even under concurrent failures.

### Resource Cleanup Chain

When a Driver is destroyed, cleanup propagates bottom-up through five levels:

1. Each **Operator** closes and releases its buffers
2. Each **OperatorContext** frees its memory allocations (propagating negative deltas up the memory tracking tree)
3. The **DriverContext** verifies zero memory and merges timing stats into its parent PipelineContext
4. The **PipelineContext** removes the Driver from its live list
5. A **DriverAndTaskTerminationTracker** counts down live Drivers; when it hits zero, the Task State Machine completes the terminal transition

## Summary: Connecting the Dots

1.  A **Query** is planned and broken into **Stages**.
2.  The Coordinator identifies the data chunks (**Splits**).
3.  The Coordinator schedules **Tasks** on specific workers to process those Splits.
4.  Inside each Task, the plan fragment is compiled into **Pipelines** — one per logical phase of the plan (scan, build, probe, output).
5.  Each Pipeline spawns **Drivers**: one per Split for scan pipelines, or a fixed count for build/exchange pipelines.
6.  A fixed-size **thread pool** runs Drivers cooperatively in **1-second quanta**, using a **multilevel feedback queue** to prioritize short queries over long ones.
7.  Each Driver's **yield loop** shuttles Pages through its Operator chain, returning a `Future` to the executor when blocked or when its quantum expires — **never blocking a thread**.
8.  As Operators produce output, data flows into **output buffers** and is pulled by downstream Tasks via **Exchanges**.
9.  When all Drivers finish, the Task enters **FLUSHING** (draining buffers), then **FINISHED** once all consumers acknowledge. Failures trigger **two-phase termination** to guarantee clean resource release.
10. The final result streams back to the Coordinator, and then to the client.
