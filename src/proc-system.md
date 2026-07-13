# [WIP] Proc System: A Separate World for Controlled Mutation

Phox separates *pure* and *procedural* worlds at the type and syntax level.

---

## Procedural types

Procedural types (`@![a]`, `Slice!`, `Ptr!`, …) represent *mutable* data structures used only inside procedural blocks.

- **cannot escape** into the pure world  
- pure functions **cannot observe** or depend on them  
- they exist only as *temporary mutable views* created via `thaw!`  
- they must be converted back to pure values via `freeze!`

Procedural types are always local to a single VM instance and never shared.

```phox
proc! {
    let xs! = thaw!(xs);     // @[a] → @![a]
    do_inplace_operation!(xs!);
    freeze!(xs!)             // @![a] → @[a]
}
```

---

## User-defined procedures

```phox
// Define procedures
// - Procedure name must end with `!`
// - Procedures are **uncurried**
// - `proc(args..) {...}` is **procedure abstraction**
*let foo! = proc(x) {...};
*let bar! = proc(x,y) {...};

// Procedure call (only allowed inside `proc! { ... }`)
proc! {
  foo!(1);
  bar!(2,3);
  ()  // `proc! {...}` must return a **pure** value
};
```

---

## Resource types

Resource types represent *opaque handles* to external OS resources.

- **can escape** into the pure world  
- pure functions **cannot observe** or pattern-match them  
- operations on resource values are allowed **only in proc world**  
- each resource type defines a destructor `drop!`  
- `drop!` is called automatically when the reference count becomes zero  
- `drop!` cannot be called explicitly

Resource values may be shared within a single VM instance without mutual exclusion,  
because procedural types never escape and pure values are immutable.

> [!NOTE]
> Resource types are **builtin** and provided only by the runtime system.  
> Users **cannot** define new resource types.  
> ```phox
> // ----
> // The following `resource ... drop! ...` syntax is **illustrative only**
> // and does not exist in the actual language:
> // ----
> // Define resource type.
> // - Resource name must end with `!`
> // - Resource type has exactly one constructor of the same name
> // - Resource type has exactly one destructor `drop!`
> // - `drop!` is called automatically when Rc becomes 0
> // - Construction and pattern match are allowed only in proc world
> resource MyResource! a = @[a]
> drop! = proc(MyResource! xs) {
>   ...
> };
> ```


---

## Concurrency and VM instances

Each VM instance is a single-threaded execution context.

- procedural types never escape → **no shared mutable state**  
- pure values are immutable → **safe to share**  
- resource values are opaque → **safe to share as long as operations are restricted to proc world**

**Only the `await` operation can transfer resource values between VM instances.**

This means:

- resource sharing/movement happens *only* at `await` boundaries  
- mutual exclusion is required *only* for resource operations that cross VM boundaries  
- no mutual exclusion is needed inside a single VM instance

If the job's return value does not contain a resource value,  
there is no limit on the number of waiters.  

Otherwise, Phox limits the number of waiters for a `JobHandle a` to `1` at most.  
In this case, `await job` consumes the resource returned by `job` (ownership
transfer), and any subsequent calls to `await job` will result in an error.

> [!NOTE]
> Resource operations may interact with external OS resources.  
> External resources are not pure and may cause race conditions.  
> Phox guarantees safety inside the VM, but external resource conflicts  
> must be handled by appropriate OS-level APIs (e.g., file locks).


