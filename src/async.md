# [T.B.D.] Async-Computation and Async I/O

An asynchronous *task* ≒ *VM instance*.  
(The Phox VM itself is a state-machine based on *Suspendable Term Reduction Abstract Machine (STReAM)*)

An asynchronous task is executed by a dedicated VM instance.  
Each VM instance is a single-threaded execution context.  
Multiple VM instances may be executed in concurrently by dedicated threads.  
(depends on scheduler implementations)

---

## User-defined tasks

**Define task constructors**:
- Task constructor name must end with `&`
- Task constructors are **uncurried**
- `task(args..) {...}` is **task constructor abstraction**

```rust , ignore
*let foo& = task(x) {...};
*let bar& = task(x,y) {...};
*let download& = task(url) {...};
```

**Constructs a task**:
- A call to a task constructor constructs a **task abstraction**.
- A task abstraction (or simply "task") of type `Task a` wraps an expression `e` of type `a`,  
  where `e` is the task constructors' body.
- The expression `e` is not evaluated immediatelly.

```rust , ignore
let tsk = download&(url);
```

**Schedule a task** and instantiate the corresponding job:

```rust , ignore
let job = schedule _DEFAULT_SCHEDULER_ tsk;
```

**Await the job completes** and take result:

```rust , ignore
let res = await job;
```

---

## Async-APIs and semantics

- `schedule : sc -> Task a -> JobHandle a`  
  : `schedule sc t`  
    - `schedule sc t` schedules task `t` to scheduler `sc`.  
      - That constructs a *dedicated VM instance* for the task `t`, and  
      - returns corresponding `job`-handle of type `JobHandle a`.  
    - The `job` is registered to the scheduler's runnable-queue, then
      - an executor will be assigned to a runnable `job`, and
      - the executor executes (start/resume) the corresponding *VM instance*.

- `await : JobHandle a -> Result Error a`  
  : `await job`  
    - If the `job` has already finished, returns `Ok val` where `val` is its resulting value of type `a`.  
    - If the `job` has already canceled, returns `Err err`.  
    - Otherwise,  
      - registers the current job to wait-queue of the `job`,  
      - and suspend the current job.  
    - When a job is finished,  
      - `Ok val` is passed to all jobs waiting in its wait-queue,  
      - and awake them. (i.e. schedule them again)  
      - the wait-queue shall be cleared.

- `cancel : JobHandle a -> ()`  
  : `cancel job`  
    - If the `job` has already finished or canceled, returns `()`.  
    - Otherwise,  
      - mark the `job` as *canceled*, then  
      - `Err err` is passed to all jobs waiting in its wait-queue,  
      - and awake them. (i.e. schedule them again)  
      - the wait-queue shall be cleared.

> [!NOTE]
> - `task`/`schedule`/`await`/`cancel` control the evaluation strategy and order of expressions.  
> - `task`/`schedule`/`await`/`cancel` themselves are *not* “operations with side effects.”  
> - Asynchronous I/O would be implemented using the proc-system, such as `task (proc! {...})`.  

---

## rough sketch

```phox
trait Cancel h a {
    cancel : h a -> ();
};

trait Await h a {
    await : h a -> a;
};

trait TryAwait h a {
    await : h a -> Result Error a;
};

trait Schedule sc a {
    schedule : sc -> Task a -> JobHandle a;
};

// -------------------------------------------------------------
impl Cancel JobHandle a {
    cancel = __cancel_job__;
};

impl TryAwait JobHandle a {
    await = __try_await_job__;
};
impl Await JobHandle a {
    await = \job. match (@{TryAwait JobHandle a}.await job) {
        Ok x => x,
        // _ => /* runtime error */
    };
};

type DefaultScheduler = @{ /* ... */ };

impl Schedule DefaultScheduler a {
    schedule = __schedule_job__;
};

let _DEFAULT_SCHEDULER_ = DefaultScheduler @{ /* ... */ };

// let t = ... ;             // t : Task a
// let job = schedule sc t;  // job : JobHandle a
// `await job |> (\Ok x. x)`   // => `@{TryAwait h a}.await` is performed
// `await job |> (\x. x + 1)`  // => `@{Await h Int}.await` is performed

```

```rust
// === Representation of Job in runtime-system. ===
type JobHandle = Rc<RefCell<Job>>;
enum Job {
    Canceled,        // => Err err
    Done(vm::Term),  // => Ok val
    InProgress {
        state: vm::State,
        waiters: Vec<JobHandle>,
    },
}
```
