# INF2007 Lecture 9 Notes — Threading, Coroutines & Flow

## 1) Processes and threads
- A process provides the resources needed to run a program.
- A process has its own virtual address space, executable code, handles, security context, identifiers, and at least one thread.
- A thread is the schedulable unit inside a process.
- Threads of the same process share memory and resources such as the heap.
- Threads still keep their own stack, registers, scheduling priority, and thread-local storage.
- Processes run in separate memory spaces; threads in the same process run in shared memory.

## 2) Basic thread creation examples
The lecture shows multiple ways to start background work:
- extend `Thread` and override `run()`
- implement `Runnable`
- `Thread { ... }.start()`
- `thread { ... }`
- `Executors.newSingleThreadExecutor()` / fixed thread pools

## 3) Android main (UI) thread
- Android app work is pushed into a queue and serviced on the main/UI thread.
- If work on the UI thread takes too long, frames can be dropped.
- Be careful with shared-memory issues such as memory contention and read/write misordering.
- UI objects must be created, updated, and destroyed only on the UI thread.
- Referencing or modifying UI objects from worker threads can cause unexpected behaviour.
- Holding Activity/UI references in long-running threads can leak the Activity after configuration changes.

## 4) Updating UI from background work
If background work needs to affect the UI, the lecture mentions:
- `runOnUiThread()`
- `Intent`
- `Handler(Looper.getMainLooper())`

Two common `Handler` approaches shown:
- `handler.post { ... }`
- `handler.sendMessage(msg)` with a `what` code

## 5) Android thread primitives
### Looper / Handler
- `Looper`: repeatedly takes tasks from a queue and executes them.
- `Handler`: manages the task queue by posting messages/runnables.
- Queue units can be `Intent`, `Runnable`, or `Message`.

### HandlerThread
- A `HandlerThread` is a `Thread` that already has a `Looper`.
- Good for several long-running operations that should execute sequentially off the UI thread.
- In the built-in example, the `Handler` must be prepared only after `start()` because the `looper` is not ready earlier.

### ThreadPoolExecutor
- Used for several intensive operations that should run in parallel.
- Handles work distribution, scheduling, load balancing, and idle-thread termination.
- Too many threads can hurt performance because threads consume memory and may exceed available CPU cores.

## 6) Coroutines
- Coroutines are a cooperative concurrency model.
- They are described as lightweight compared with threads.
- They help move long-running work off the main thread and support cleaner async code.
- Structured concurrency improves cancellation and exception handling.

## 7) Coroutine builders
### `launch`
- Fire-and-forget.
- Similar role to starting a thread.
- Returns a `Job`.

### `async`
- Used when a result is needed later.
- Returns a `Deferred`.
- Use `.await()` to get the result.

### `runBlocking`
- Bridges blocking code and suspending code.
- Creates a coroutine scope and blocks the current thread.

## 8) Suspend functions
- `delay()` is a suspend function: it suspends the coroutine without blocking the underlying thread.
- Suspend functions can be called only from another suspend function or from a coroutine.
- A suspend function does not automatically create a coroutine or provide a `CoroutineScope`.
- Use `coroutineScope {}` inside suspend code when a child scope is needed.
- Suspend functions make async flows read more like sequential code and help avoid callback hell.

## 9) Sequential vs parallel work
- Sequential suspend calls run one after another by default.
- To overlap independent work, use `async` and await both results.
- The lecture example reduces total time from about `2000+ ms` to `1000+ ms` when two 1-second tasks run concurrently.

## 10) Dispatchers
### `Dispatchers.Main`
- Main Android thread.
- UI work and quick tasks.

### `Dispatchers.IO`
- Disk and network I/O.
- File access, Room, network requests.

### `Dispatchers.Default`
- CPU-intensive work.
- Sorting, parsing JSON, heavy computation.

### `withContext(...)`
- Switches coroutine context without starting a brand-new coroutine.
- Typical pattern:
  - get data on `Dispatchers.IO`
  - process it on `Dispatchers.Default`
  - resume and update UI on `Dispatchers.Main`

## 11) Structured concurrency
- Focuses on coroutine hierarchy and lifetime management.
- Every coroutine belongs to a `CoroutineScope`.
- `CoroutineContext` can include:
  - `Job`
  - `CoroutineDispatcher`
  - `CoroutineName`
  - `CoroutineExceptionHandler`
- Parent coroutines wait for their children.
- Parent-child relation can be overridden by:
  - launching in another scope such as `GlobalScope`
  - passing a different `Job`

## 12) Lifecycle-aware coroutine scopes
### `viewModelScope`
- One per `ViewModel`.
- Coroutines are cancelled automatically when the `ViewModel` is cleared.

### `lifecycleScope`
- One per `Lifecycle` owner.
- Coroutines are cancelled when the lifecycle is destroyed.

## 13) Parent and child coroutine behaviour
- Parent cannot fully complete before children complete.
- If a child finishes first, parent continues.
- If parent is cancelled, children are cancelled.
- If child is cancelled, parent continues.
- Child exceptions normally cancel parent and siblings unless supervision is used.

## 14) Coroutine cancellation
- Cancellation is cooperative, not forced.
- Cancellation works at cancellation points such as suspend functions.
- Useful APIs mentioned:
  - `job.cancel()`
  - `scope.cancel()`
  - `job.join()`
  - `job.cancelAndJoin()`
  - `withTimeout(...)`
  - `isActive`
  - `ensureActive()`
  - `yield()`
- `SupervisorJob()` changes cancellation so it propagates downward only.

## 15) Race conditions
- Shared mutable state like `counter++` can produce incorrect results when many coroutines modify it concurrently.
- The lecture mentions that simple volatile-style thinking is not enough.
- Safer options shown:
  - `AtomicInteger`
  - `Mutex` with `withLock`
  - `Actor`
- The slide also notes that `synchronized {}` is thread-style locking and does not support suspend functions directly.

## 16) Flow
- Flow is a stream of values that can be computed asynchronously.
- It can emit multiple values sequentially.
- It fits reactive programming.
- Collect a flow from a coroutine, for example in `lifecycleScope.launch { flow.collect { ... } }`.
- Other creation helpers mentioned: `flowOf()` and `asFlow()`.

### Flow vs LiveData
- Flow supports thread switching and back-pressure.
- Flow is part of Kotlin coroutines and integrates well with coroutine code.
- Flow is cold and lazy.
- LiveData producers do not depend on active collection the same way cold flows do.

### Flow operators listed
- `map()`
- `filter()`
- `onEach()`
- `flowOn()`
- `toList()`
- `sample()`
- `reduce()`
- `fold()`
- `flatMapConcat()`
- `flatMapMerge()`
- `zip()`
- `buffer()`
- `first()`
- `collectLatest()`
- `take()` / `takeWhile()`
- `drop()` / `dropWhile()`
- `onCompletion()`

### Flow context
- By default, the producer inside `flow {}` runs in the collector’s coroutine context.
- To change upstream context, use `flowOn(...)`.

## 17) SharedFlow and StateFlow
### SharedFlow
- `Flow` is cold; `SharedFlow` is hot.
- Hot streams emit regardless of observers.
- It supports one-to-many sharing.
- `MutableSharedFlow(...)` parameters shown:
  - `replay`
  - `extraBufferCapacity`
  - `onBufferOverflow`
- Default `replay` is `0`.
- `shareIn()` can convert a cold flow into shared flow.

### StateFlow
- Special variant of `SharedFlow`.
- Closest to `LiveData` in the lecture.
- Always has a value.
- Value is unique/current state.
- Replays only the latest value to collectors.
- Supports multiple observers.

## 18) Compose side effects
### `LaunchedEffect`
- Safe way to call suspend functions from a composable.
- Launches when it enters composition.
- Cancelled when it leaves composition.
- If keys change, the old coroutine is cancelled and a new one starts.

### `rememberCoroutineScope()`
- Returns a `CoroutineScope` tied to that composition location.
- Good for launching coroutines from event handlers such as button clicks.
- Automatically cancelled when that part of the composition leaves.

### `SideEffect`
- Used to publish Compose state to non-Compose code.
- Runs after every successful recomposition.

### `DisposableEffect`
- Used for effects that require cleanup.
- Re-runs when keys change.
- Must include `onDispose { ... }`.
- Typical use: add a lifecycle observer, then remove it on disposal.

## 19) High-yield revision checklist
Make sure you can explain:
1. process vs thread
2. why UI updates must stay on the main thread
3. Looper / Handler / HandlerThread roles
4. HandlerThread vs ThreadPoolExecutor use case
5. `launch` vs `async` vs `runBlocking`
6. suspend function behaviour
7. `Dispatchers.Main` vs `IO` vs `Default`
8. `withContext(...)`
9. structured concurrency and parent-child cancellation
10. `viewModelScope` vs `lifecycleScope`
11. cooperative cancellation
12. race condition fixes: Atomic / Mutex / Actor
13. cold Flow vs hot SharedFlow / StateFlow
14. `LaunchedEffect`, `rememberCoroutineScope`, `SideEffect`, `DisposableEffect`
