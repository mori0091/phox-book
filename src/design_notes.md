# Design Notes

## Phox Type System (Future)

| (kind)   | variable           | constraint solver           | constraint         | var-gen / union-find | constraint set                     |
|----------|--------------------|-----------------------------|--------------------|----------------------|------------------------------------|
| ~~Kind~~ | ~~KindVarId `$a`~~ | ~~KindSolver~~              | ~~KindConstraint~~ | ~~KindVarContext~~   | ~~KindConstraintSet~~              |
| Type     | TypeVarId `a`      | UnifiedSolver (Type + MPTC) | TypeConstraint     | TypeContext          | UnifiedConstraintSet (Type + MPTC) |
| Row      | RowVarId  `@a`     | UnifiedSolver (Row)         | RowConstraint      | RowContext           | UnifiedConstraintSet (Row)         |
| Nat      | NatVarId  `#a`     | UnifiedSolver (Nat)         | NatConstraint      | NatContext           | UnifiedConstraintSet (Nat)         |

> [!NOTE]
> TODO: Change section-syntax from `#(op x)` to `|(op x)`.
> 1. [X] Add new syntax `|(op x)` and mark old syntax as **deprecated**.
> 2. [ ] After `Nat` solver is implemented, remove old syntax `#(op x)`.

> [!NOTE]
> - `$a`, `@a`, and `#a` are primarily used in pretty-printed inferred types.
> - KindVarId `$a` is internal use only for kind inference.
> - In user code,
>   - type-level variables **in expressions** should generally be written as plain identifiers like `a`.
>   - type-level variables **in `type`/`trait` declarations** should be written as `a` and `#a`
>     for Type and Nat variables respectively.
> - As a future extension, `?a`, `#a`, and `@a` may be accepted as syntactic sugar **in expressions** for
>   explicit kind annotations `(a : Type)`, `(a : Nat)`, and `(a : Row)` respectively.

> [!NOTE]
> Maybe `Nat` kind will not be implemented...  
> it's not so useful.
 
## Rough sketch of Phox Type System process-pipeline

1. parse: source code "a" → RawAST such as RawTyVar("a")
2. resolve phase: RawTyVar("a") → RowVarId(r), NatVarId(n), or TypeVarId(t)
  - if `a` appeared in Row context (`@{ .. | a }`), `a` shall be RowVarId(r)
  - if `a` appeared in type parameters (`@{T a}`), `a` shall be NatVarId(n) or TypeVarId(t)
    - check `type`/`trait` declarations and its parameters' kind,
    - then assign NatVarId(n) or TypeVarId(t) according to the kind.
3. infer phase: infer and solve with UnifiedSolver
4. apply phase: bake specialized code

> [!NOTE]
> - row-var appears only in record/row contexts such as `@{...}`, so no ambiguity arises.
> - Since both type-var and nat-var can appear as type parameters only in trait record `@{T a b}`,
>   they must resolve to either NatVarId or TypeVarId
>   after kind-check and kind-assignment in resolve phase.
> - Kind inference and KindSolver are not needed for Phox.

## Rough sketh of Phox Type System data sturctures

``` rust
// Constructors for Kind expression
pub enum Kind {
    Fun(Box<Kind>, Box<Kind>),   // κ1 -> κ2
    Type,                        // τ
    Row,                         // ρ
    // Nat,                      // ν
}

pub enum TypeExpr {
    Ty(Type),
    Row(Row),
    // Nat(Nat),
}

// pub fn kind_of(texpr: &TypeExpr) -> Kind {
//     match texpr {
//         TypeExpr::Ty(ty) => match ty {
//             Type::Var(_) => {
//                 todo!()
//             }
//             Type::Con(_name) => {
//                 todo!()
//             }
//             Type::App(ref t1, ref t2) => {
//                 Kind::Fun(Box::new(kind_of(t1)), Box::new(kind_of(t2)))
//             }
//             Type::Fun(_, _) | Type::Tuple(_) | Type::Record(_) => {
//                 Kind::Type
//             }
//         }
//     }
// }

// Constructors for Type expression
pub enum Type {
    Var(TypeVarId),             // 型変数
    Fun(Box<Type>, Box<Type>),  // 関数型
    Con(Symbol),                // 型構築子

    App(Box<TypeExpr>, Box<TypeExpr>),  // 型適用

    Tuple(Vec<Type>),
    // Record(Vec<(String, Type)>), // => Type::App(Type::Con("@{}"), TypeExpr::Row(row))
}

pub struct TypeVarId(usize);  // `a`, `?100`

pub enum TypeConstraint {
    TypeEq(Type, Type),
    // -- for MPTC (`trait`/`impl`) and Overloaded Function Template (`*let`) --
    TraitBound(TraitHead),
    Overloaded(Symbol, Type, Vec<SchemeTemplate<Type>>),
}

// Constructors for Row expression
pub enum Row {
    Empty,                              // `@{}`
    Var(RowVarId),                      // `@{ | row }`
    Ext(Box<(Label, Type)>, Box<Row>),  // `@{ x:τ | row }`
}

pub struct RowVarId(usize);  // `@a`, `@?100`
pub type Label = String;

// T.B.D.
pub enum RowConstraint {
    RowEq(Row, Row),
    Has(Row, Label),
    Lacks(Row, Label),
}

// Constructors for Nat expression
// pub enum Nat {
//     Lit(usize),                         // `0`, `1`, ...
//     Var(NatVarId),                      // `n`
//     Add(Box<Nat>, Box<Nat>),            // `n + m`
// }

// pub struct NatVarId(usize);  // `#a`, `#?100`

// T.B.D.
// pub enum NatConstraint {
//     NatEq(Nat, Nat),
//     NatLT(Nat, Nat),
//     // NatEqu(...),   // natural number equation (e.g. `m + n = 0`)
// }

```
``` rust
pub struct Scheme<T> {
    pub vars: Vec<Var>,             // quantified variables              (ex. `∀ a.`)
    pub constraints: ConstraintSet, // constraints / trait bounds        (ex. `(Eq a, Ord a) =>`)
    pub target: T,                  // the type, or                      (ex. `a -> a -> Bool`)
                                    // the trait head produced by `impl` (ex. `Eq (List a)`)
}

pub enum Var {
    Ty(TypeVarId),
    Row(RowVarId),
    // Nat(NatVarId),
}

pub enum Constraint {
    Ty(TypeConstraint),
    Row(RowConstraint),
    // Nat(NatConstraint),
}

pub struct ConstraintSet {
    pub primary:  Option<Box<TraitHead>>,
    pub requires: BTreeSet<Constraint>,
}

pub struct UnifiedSolver {
    // ...
}
```

