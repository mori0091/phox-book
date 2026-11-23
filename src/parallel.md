# [T.B.D.] Parallel computation

- Task
  : An unevaluated expression.  
    A task is a pure value.  
    It does not execute by itself.  
    A task becomes a running job when spawned by `join` or `par`.

- Job
  : An executing entity that evaluates a task in a dedicated context.  
    Jobs are the core abstraction of concurrency and scheduling, in Run-Time System.

- `task expr`
  : Constructs a task of `Task a` type where `expr` is an expression of `a` type.  
    The `Task a` represents an unevaluated expression of `a` type.

- `join t1 t2`
  : Executes tasks `t1` and `t2` in parallel.  
    Returns a tuple of both results.

- `par xs`
  : Executes each tasks of `Task ()` type in the array `xs` in parallel.  
    Returns `()`.

```rust , ignore
task (1 + 2)
// => <task> : Task Int

join
    (task (1 + 2))
    (task ("abc" ++ "de"))
// => (3, "abcde") : (Int, ScalarString)

par @[
    task e1,  // : (Task ())
    task e2,  // : (Task ())
    task e3,  // : (Task ())
]
// => () : ()
```
