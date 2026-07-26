# [WIP] Structural Transparency of Types (STraT)

*Structural Transparency of Types (STraT)* is an attribute of types.

---

## Definition

```rust , ignore
enum StructuralTransparency {
  Opaque,                             // the type is opaque
  SemiTransparent { ts: Vec<Type> },  // transparency of the type depends on transparency of `ts`.
  Transparent,                        // the type is transparent
  Any,                                // transparency is not determined (for fresh type variable)
}
```

- Function types (closures) are `Opaque`.
- Primitive types are `Transparent`.
- Procedural types; such as dynamic arrays; are `Opaque`.  
  (Meaningless because they cannot escape to pure world and cannot cross the VM boundary)
- Resource types; such as file-handle; are `Transparent`.  
  (Though its value structure is opaque, the run-time system ensures that it contains no other resources nor closures)
- Tuples, Arrays, Records are:
  - `Opaque` if an element type was `Opaque`,
  - `Transparent` if all element types were `Transparent`,
  - `SemiTransparent` otherwise.
- ADTs are:
  - `Opaque` if an element of any variant was `Opaque`,
  - `Transparent` if all element types of all variants were `Transparent`,
  - `SemiTransparent` otherwise.
- Fresh type variables are `Any`.  
  (Their transparency is determined via type unification process)

---

## Example

- `a` is `Any` (if not unified yet)
- `a -> b` is `Opaque`.
- `Int` is `Transparent`.
- `File!` is `Transparent`. (resource types)
- `Option Int` is `Transparent`.
- `Option File!` is `Transparent`.
- `Option a` is `SemiTransparent { ts: vec![a] }`.
- `Result e a` is `SemiTransparent { ts: vec![e, a] }`.
- `Map s a b` is `Opaque`. (because its data constructor is `Map (a -> b) (s a)`)

---

## Unification

If type `t1` and `t2` are successfully unified (`unify(t1, t2)` succeeded),  
their *STraT* attributes are merged.

- `merge(X, Opaque) = Opaque`
- `merge(Opaque, X) = Opaque`
- `merge(Transparent, Transparent) = Transparent`
- `merge(Transparent, SemiTransparent{A}) = SemiTransparent{A}`
- `merge(SemiTransparent{A}, Transparent) = SemiTransparent{A}`
- `merge(SemiTransparent{A}, SemiTransparent{B}) = SemiTransparent{A ∪ B}`
- `merge(X, Any) = X`
- `merge(Any, X) = X`

In other words, the unification of *STraT* corresponds to
the maximum (join) of the following partially ordered set (poset):
```
Opaque > SemiTransparent > Transparent > Any
```

---

## Type Constraints

- `ResourceFree` = “No resource value”
- `ResourceTransparent` = “No opaque values that hide resources”

By definition, a resource type is **transparent as a type** but **opaque as a value**.

In contrast, `ResourceFree` and `ResourceTransparent` are type-constraints that  
ensure the type system can reliably check for **transparency of resource ownership**.


### `ResourceFree` type-constraint

The constraint `ResourceFree(ty)` is:

- if `ty` was `Opaque`:  
  - `ResourceFree(ty)` causes an error.
- if `ty` was `Transparent`:
  - `ResourceFree(ty)` causes an error, if the `ty` itself or its type-parameters contain resource types.
  - `ResourceFree(ty)` is OK, otherwise.
- if `ty` was `SemiTransparent{ts: vec![a, b, ...]}`:
  - `ResourceFree(ty)` causes an error, if the `ty` itself or its type-parameters contain resource types.
  - `ResourceFree(ty)` is `ResourceFree(a) ∧ ResourceFree(b) ∧ ...`, otherwise.


### `ResourceTransparent` type-constraint

The constraint `ResourceTransparent(ty)` is:

- if `ty` was `Opaque`:  
  - `ResourceTransparent(ty)` causes an error.
- if `ty` was `Transparent`:
  - `ResourceTransparent(ty)` is OK.
- if `ty` was `SemiTransparent{ts: vec![a, b, ...]}`:
  - `ResourceTransparent(ty)` is `ResourceTransparent(a) ∧ ResourceTransparent(b) ∧ ...`.


