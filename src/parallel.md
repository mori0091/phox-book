# [T.B.D.] Parallel computation

`par { ... }` evaluates expressions in parallel and returns a tuple of results.

```rust , ignore
let (x, y, z) = par { e1, e2, e3 };
```

`race { ... }` evaluates expressions in parallel and returns the first result obtained.

```rust , ignore
let x = race { e1, e2, e3 };
```

