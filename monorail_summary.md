# C++ Model System Design

## 1. Purpose and Scope

This repository implements a C++ discrete-event simulation model for hardware-like designs. The model represents clocks, resets, combinational logic, sequential logic, memories, and testbenches as processes scheduled by a simulation kernel.

The primary design goal is to preserve hardware-style scheduling semantics while allowing independent processes to execute concurrently. The design therefore separates:

- simulation time from wall-clock time;
- scheduling regions from C++ thread execution;
- process state from signal storage;
- event triggering from process execution;
- model-specific modules from kernel services.

The implementation is visible mainly in `common/common.h`, `common/common.cpp`, and the example models under `challenge_lfsr/` and `riscv-simple-sv/pipeline_cpp_model/`.

## 2. Main Components

### 2.1 SimulationKernel

`SimulationKernel` owns the global simulation state:

- the scheduled event map, indexed by simulation time and sub-slot;
- all global `SignalInfo` objects;
- region event runners;
- delayed events;
- the optional thread pool;
- the current simulation time and iteration number;
- execution statistics.

It is exposed as a singleton through `SimulationKernel::createKernel()` and `SimulationKernel::kernel()`.

The kernel is the authoritative owner of simulation time and is responsible for advancing the simulation only after the current time slot and region have completed.

### 2.2 Signal and SignalInfo

`Signal` stores a logic vector and edge state. A signal can contain `ZERO`, `ONE`, `X`, or `Z` values. `SignalInfo` associates a signal with triggered processes:

- whole-signal triggers;
- bit-indexed triggers;
- edge-specific trigger maps.

Signals are accessed globally by integer signal ID. Modules store signal IDs rather than owning signal values directly.

### 2.3 Module

`Module` is a lightweight structural base class. It stores signal IDs associated with a model object. Derived modules such as `DFlipFlop`, `MiniM10`, `M10`, `VariableM`, and `Testbench` build the model hierarchy and register processes with the kernel.

Modules generally own their process objects through `std::optional` and own dynamically allocated child modules through vectors of pointers.

### 2.4 Event

`Event` is the base scheduling abstraction. It provides:

- event execution;
- signal updates;
- trigger lookup and propagation;
- optional local signal contexts;
- per-kernel-iteration deduplication.

`Event::run()` normally calls `preProcess()` and then `execute()`. `preProcess()` uses the event's atomic iteration value to prevent the same event from executing more than once in one normal-flow kernel iteration.

### 2.5 ProcessEvent

`ProcessEvent` extends `Event` with coroutine support. Model processes implement:

```cpp
ReturnObject run() override;
```

The process body can suspend with `co_await Awaiter(...)`. `ProcessEvent` stores the coroutine handle and resumption time so the kernel can resume the process in a later time slot or region.

### 2.6 EventRunner

`EventRunner` contains the events associated with one execution batch. It can contain:

- directly scheduled events;
- pointers to triggered-process vectors;
- dynamically appended events in triggered-now mode.

It runs a region and acts as the unit of work assigned to the thread pool.

### 2.7 ThreadPool

`ThreadPool` creates worker threads when the requested thread count is greater than zero. Each worker receives the same `EventRunner` pointer and processes a deterministic index partition such as:

```text
worker 0: event 0, N, 2N, ...
worker 1: event 1, N+1, 2N+1, ...
```

The kernel waits until all workers report completion before advancing. The default Makefile configuration uses zero worker threads, which means single-threaded execution.

## 3. Event Lifecycle

The normal lifecycle is:

1. The kernel selects the earliest scheduled simulation time.
2. It moves the corresponding `EventRunner` out of the scheduled map.
3. It updates the current simulation time.
4. It runs the runner for the current sub-slot.
5. It drains zero-delay region events in region order.
6. It moves delayed events into the scheduled map.
7. It selects the next time slot.

When an event changes a signal, the kernel determines the signal edge and looks up the registered processes. Those process vectors are either:

- added to the triggered region in normal mode; or
- appended to the currently executing runner in `RUN_TRIGGERED_NOW` mode.

## 4. Simulation Regions

The code defines these regions:

```text
ACTIVE = 0
UPDATE_GLOBAL = 1
TRIGGERED = 2
INACTIVE = 3
NBA = 4
```

`SimulationKernel::runNextRegion()` scans regions in ascending order and runs the first non-empty region. This produces the current effective ordering:

```text
ACTIVE -> UPDATE_GLOBAL -> TRIGGERED -> INACTIVE -> NBA
```

The regions are an implementation-level scheduling abstraction inspired by hardware simulation regions. They are not yet a complete implementation of the SystemVerilog event-region standard.

### ACTIVE

Use for ordinary process execution and testbench activity.

### UPDATE_GLOBAL

Use as a commit phase for combinational calculations. A combinational process can:

1. read its inputs;
2. calculate local result values;
3. suspend at zero delay into `UPDATE_GLOBAL`;
4. publish its result after resumption.

This creates a separation between calculation and publication. It does not itself serialize concurrent C++ writes.

### TRIGGERED

Used in normal mode for processes triggered by signal changes. Triggered process vectors are collected and executed after the current event batch.

### INACTIVE

Defined but not substantially used by the current examples.

### NBA

Used for nonblocking, clocked state updates. Flip-flop processes commonly sample inputs before the await and update their state after resuming in `NBA`.

## 5. Coroutine Design

The custom `Awaiter` takes a process, delay, and region. On suspension, it stores the coroutine handle in the process and registers the process with either:

- the delayed-event queue, when the delay is nonzero; or
- a region runner, when the delay is zero.

Example:

```cpp
Signal sampledInput = getSignal(inputID);
co_await Awaiter(this, 0, Region::NBA);
updateSignal(outputID, sampledInput);
```

The coroutine frame preserves local variables across suspension. A copied `Signal`, `LogicValue`, or computed result is stable across the await. A reference such as `Signal&` continues to refer to live shared signal storage and can observe a later value after resumption.

The safe process pattern is therefore:

```text
copy inputs
compute locally
await
publish one explicit result
```

## 6. Threading and Synchronization

The code uses these synchronization mechanisms:

### Atomic values

- `Event::d_iteration`: deduplicates normal-mode event execution within one kernel iteration.
- `ThreadInfo::d_eventRunner`: publishes the current runner to each worker.
- `ThreadInfo::d_processCount`: collects worker statistics.
- `ThreadPool::d_pendingThreads`: tracks completion of the current runner.
- `ThreadPool::d_stop`: stops worker loops.

### Mutex and condition variable

`d_mutex` and `d_cv` are used by `notifyNewWork()`, but worker threads currently spin using `yield()` rather than waiting on the condition variable. This means the condition variable is an incomplete or unused optimization path in the current implementation.

### Region barrier

The main useful synchronization behavior is the worker completion barrier. The kernel waits for `d_pendingThreads` to reach zero before it advances to later scheduling work. This prevents one region from being advanced while workers are still executing another region.

## 7. Determinism Model

The current design provides partial determinism:

- simulation time advances in a single kernel thread;
- regions are processed in a fixed order;
- workers complete before the next region is processed;
- normal-mode duplicate execution is suppressed per kernel iteration;
- the event partition is based on stable vector indices.

The design does not currently guarantee deterministic ordering inside a parallel region.

### First-trigger behavior

In normal mode, if the same process is placed into an event batch more than once, `d_iteration.exchange(...)` allows the first invocation to reach the atomic exchange to execute. In single-threaded mode this is normally the first vector occurrence. In multithreaded mode it is the first worker to reach the atomic operation, which is not necessarily the lowest event index.

Later triggers for that process in the same iteration are discarded. This is a deduplication policy, not a deterministic last-writer or first-logical-trigger policy.

### Signal writes

Global signal update methods directly modify shared `Signal` objects. They are not protected by a signal-level mutex and are not collected into a deterministic commit queue.

Therefore, the implementation currently assumes one writer per signal, or writers that update disjoint bits with no conflicting whole-signal operations. This assumption is not enforced by the type system or kernel.

If two workers write the same signal, final state can depend on wall-clock scheduling. Even disjoint-bit writes require care because `Signal` also tracks edge state and because packed representations may update multiple underlying bits.

Locks would prevent some memory races but would not provide deterministic winner selection. Deterministic behavior requires either:

- strict single-writer ownership;
- immutable input snapshots and buffered output updates;
- a barrier followed by ordered serial commits; or
- an explicitly defined deterministic merge policy.

The strongest general model is:

```text
parallel read/compute
        -> barrier
        -> sort updates by stable process/signal key
        -> serial commit
        -> trigger dependent processes
```

## 8. Local Signal Contexts

`LocalSignalsContext` is intended to provide event-local cached signal values, particularly for `RUN_TRIGGERED_NOW` mode.

The mechanism supports:

- copying a global signal into an event-local cache on first access;
- local updates before an await;
- dirty whole-signal or range tracking;
- cloning and merging local contexts into triggered processes;
- committing local values back to global signals.

This isolates intermediate process results from global storage, but it is not a complete conflict-resolution system. Two local contexts can still commit conflicting values. The current commit path does not impose a stable ordering or reject conflicting writes.

Some models, including `challenge_lfsr/lfsr6.cpp`, access signals directly with `SimulationKernel::kernel().getSignal(...)` and update them with `updateSignal(...)`. Those paths bypass the local cache.

## 9. Use of TBB `concurrent_vector`

The alias:

```cpp
using cvector = tbb::concurrent_vector<T>;
```

is used for event and trigger lists. It supports concurrent append operations and avoids ordinary `std::vector` reallocation hazards when events are added while a runner is active.

It does not guarantee:

- deterministic append order;
- deterministic process execution order;
- atomic signal updates;
- conflict resolution;
- stable ordering based on process identity.

The container solves concurrent collection growth, not simulation semantics.

## 10. Design Patterns Present

### Discrete-event simulation

The kernel advances from event time to event time rather than executing a fixed real-time loop.

### Singleton

`SimulationKernel` and `Timer` use static singleton pointers. This simplifies global access but increases coupling and makes multiple independent simulations difficult.

### Observer / publish-subscribe

Signals maintain trigger registrations. A signal update notifies processes registered for whole-signal, bit-level, or edge-specific changes.

### Event queue

Scheduled and delayed events are stored in time-indexed queues and region runners.

### Reactor-like event loop

The kernel repeatedly selects work, executes it, processes resulting events, and advances to the next time slot.

### Strategy-like execution modes

Normal flow and `RUN_TRIGGERED_NOW` alter event dispatch, local caching, and process handling. Currently this behavior is selected by a global environment variable rather than an explicit strategy object.

### Composite model hierarchy

Modules contain child modules and process objects. For example, `VariableM` contains `M10` instances, which contain flip-flops and MiniM10 instances.

### Template alias / adapter

`cvector` adapts TBB's concurrent container behind a project-local type name.

## 11. SOLID Assessment

### Single Responsibility Principle

Partially satisfied.

Good separation exists between:

- kernel scheduling;
- event execution;
- signal representation;
- thread-pool workers;
- model modules.

However, `SimulationKernel` also owns signal storage, scheduling, statistics, formatted display, file loading, and process-mode reporting. `Event` combines execution lifecycle, signal access, trigger propagation, and local-cache management.

Recommended improvement: extract signal storage, trigger dispatch, statistics, and system I/O behind separate services.

### Open/Closed Principle

Partially satisfied.

New modules and process types can be added through `Module` and `ProcessEvent` inheritance. New scheduling modes are harder because behavior is distributed through global `RUN_TRIGGERED_NOW` checks throughout the implementation.

Recommended improvement: encapsulate dispatch behavior behind an execution-policy interface.

### Liskov Substitution Principle

Mostly satisfied for `Event` and `ProcessEvent`, but the contract is subtle. `ProcessEvent` has coroutine-specific lifecycle rules, while `Event` has ordinary execution rules. Derived processes must respect hidden assumptions about `run()`, `execute()`, resumption, and cleanup.

Recommended improvement: document the lifecycle contract and separate coroutine events from ordinary update events more explicitly.

### Interface Segregation Principle

Weak.

`Event` and `ProcessEvent` expose many update, cache, commit, integer-conversion, scheduling, and coroutine methods. Most model processes use only a small subset.

Recommended improvement: provide narrower interfaces such as `SignalReader`, `SignalWriter`, `Scheduler`, and `CoroutineProcessContext`.

### Dependency Inversion Principle

Weak to partial.

Model code depends directly on the global singleton kernel and concrete signal IDs. The kernel depends directly on TBB containers and thread behavior. This is convenient for a prototype but makes testing and alternative schedulers difficult.

Recommended improvement: inject a simulation context or scheduler interface into processes and modules. Keep TBB behind a scheduler/container abstraction.

## 12. Main Risks

1. **Concurrent signal mutation:** global `Signal` objects are not protected by deterministic commit logic.
2. **Reference lifetime across await:** references to live signals can observe changed values after suspension.
3. **Nondeterministic first trigger:** atomic deduplication selects the first worker to reach the exchange.
4. **Concurrent trigger ordering:** `cvector` supports growth but does not provide semantic ordering.
5. **Incomplete triggered-now semantics:** event insertion and current-runner handling differ between single-threaded and thread-pool paths.
6. **Global singleton coupling:** model code is tightly coupled to one active kernel.
7. **Manual ownership:** child modules are allocated with `new` and deleted manually.
8. **Region model incompleteness:** the region enum is useful but does not fully model all SystemVerilog scheduling regions.
9. **Statistics and simulation concerns are mixed:** reporting code is embedded in the kernel execution service.
10. **Unimplemented synchronization path:** the condition variable exists, but workers currently spin instead of waiting.

## 13. Recommended Evolution Path

### Priority 1: Make signal ownership explicit

Choose and enforce one of these contracts:

- one process owns each signal;
- one process owns each signal bit range;
- all writes are buffered and committed by the kernel;
- conflicting writes are rejected deterministically.

For a general simulator, buffered commits are the most flexible choice.

### Priority 2: Introduce immutable region snapshots

At the beginning of a region, workers should read from an immutable snapshot. A process should not read a live global signal after an await unless that behavior is explicitly intended.

### Priority 3: Add deterministic commit ordering

Assign every process a stable ID. Collect output updates and apply them in a fixed order such as:

```text
simulation time -> region -> process ID -> signal ID -> bit/range
```

Trigger downstream processes only after the commit phase completes.

### Priority 4: Separate execution policies

Replace scattered environment-variable checks with an explicit policy object for normal mode and triggered-now mode.

### Priority 5: Improve ownership and testability

Use RAII containers such as `std::unique_ptr`, inject kernel/scheduler dependencies, and split kernel services into smaller classes.

### Priority 6: Add determinism tests

Run the same model repeatedly with multiple thread counts and compare:

- final signal state;
- per-time-slot outputs;
- trigger counts;
- region traces.

A deterministic simulator should produce identical logical results regardless of worker count and worker scheduling.

## 14. Overall Evaluation

This is a capable prototype architecture for experimenting with hardware-model simulation and parallel event execution. Its strongest ideas are the explicit simulation kernel, region-based scheduling, coroutine suspension, trigger registration, and worker completion barrier.

Its central limitation is that scheduling synchronization and signal-state synchronization are separate concerns, but only the former is currently implemented. The thread pool can ensure that a region has finished, yet it cannot ensure that competing signal updates within the region have a deterministic result.

The project should therefore be described as:

> A coroutine-based discrete-event hardware simulator with region barriers and optional parallel event execution, currently relying on an implicit single-writer signal discipline rather than enforcing deterministic parallel signal commits.
 