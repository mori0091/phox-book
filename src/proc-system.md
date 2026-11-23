# [WIP] Proc System: A Separate World for Controlled Mutation

Phox separates *pure* and *procedural* worlds at the type and syntax level.

---

## User-defined procedures

**Define procedures**:
- Procedure name must end with `!`
- Procedures are **uncurried**
- `proc(args..) {...}` is **procedure abstraction**

```rust , ignore
*let foo! = proc(x) {...};
*let bar! = proc(x,y) {...};
*let download! = proc(url) {...};
```

**Procedure call** (only allowed inside procedure or `proc! { ... }`):
```rust , ignore
proc! {
  foo!(1);
  bar!(2,3);
  ()  // `proc! {...}` must return a **pure** value
};

let x = proc! { download!(url) };
```

