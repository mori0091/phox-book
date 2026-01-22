# Design Notes

## Phox Type System (Future)

| (sort) | variable       | constraint solver           | constraint     | var-gen / union-find | constraint set                     |
|--------|----------------|-----------------------------|----------------|----------------------|------------------------------------|
| Sort   | SortVarId `$a` | SortSolver                  | SortConstraint | SortVarContext       | SortConstraintSet                  |
| Type   | TypeVarId `a`  | UnifiedSolver (Type + MPTC) | TypeConstraint | TypeContext          | UnifiedConstraintSet (Type + MPTC) |
| Row    | RowVarId  `@a` | UnifiedSolver (Row)         | RowConstraint  | RowContext           | UnifiedConstraintSet (Row)         |
| Nat    | NatVarId  `#a` | UnifiedSolver (Nat)         | NatConstraint  | NatContext           | UnifiedConstraintSet (Nat)         |

> [!NOTE]
> TODO: Change section-syntax from `#(op x)` to `|op x|`.
> 1. Add new syntax `|op x|` and mark old syntax as **deprecated**.
> 2. After `Nat` solver is implemented, remove old syntax `#(op x)`.

> [!NOTE]
> - `$a`, `@a`, and `#a` are primarily used in pretty-printed inferred types.
> - In user code, variables should generally be written as plain identifiers like `a`.
> - As a future extension, `#a` and `?a` may be accepted as syntactic sugar for
>   explicit sort annotations `(a : Nat)` and `(a : Type)` respectively.
 
``` rust
// Constructors for Sort expression (Sort is something lika a super-type of Kind)
pub enum Sort {
    Fun(Box<Sort>, Box<Sort>),   // s1 -> s2
    Type,                        // τ
    Row,                         // ρ
    Nat,                         // ν
}

pub struct SortVarId(usize);  // `$a`, `$?100`

pub enum SortConstraint {
    SortEq(Sort, Sort),
}

pub enum TypeExpr {
    Type { ty: Type },
    Row { row: Row },
    Nat { nat: Nat },
}

impl TypeExpr {
    pub fn sort_of(&self) -> Sort {
        todo!();
    }
}

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
pub enum Nat {
    Lit(usize),                         // `0`, `1`, ...
    Var(NatVarId),                      // `n`
    Add(Box<Nat>, Box<Nat>),            // `n + m`
}

pub struct NatVarId(usize);  // `#a`, `#?100`

// T.B.D.
pub enum NatConstraint {
    NatEq(Nat, Nat),
    NatLT(Nat, Nat),
    // NatEqu(...),   // natural number equation (e.g. `m + n = 0`)
}

```
``` rust
pub struct Scheme<T> {
    pub vars: Vars,                        // quantified variables              (ex. `∀ a.`)
    pub constraints: UnifiedConstraintSet, // constraints / trait bounds        (ex. `(Eq a, Ord a) =>`)
    pub target: T,                         // the type, or                      (ex. `a -> a -> Bool`)
                                           // the trait head produced by `impl` (ex. `Eq (List a)`)
}

pub struct Vars {
    pub sort: Vec<SortVarId>,
    pub row:  Vec<RowVarId>,
    pub nat:  Vec<NatVarId>,
    pub ty:   Vec<TypeVarId>,
}

pub struct UnifiedConstraintSet {
    pub requires_row: BTreeSet<RowConstraint>,
    pub requires_nat: BTreeSet<NatConstraint>,
    pub primary_ty:   Option<Box<TraitHead>>,
    pub requires_ty:  BTreeSet<TypeConstraint>,
}

pub struct UnifiedSolver {
    // ...
}
```

``` rust
pub struct SortConstraintSet {
    pub requires_sort: BTreeSet<SortConstraint>,
}

pub struct SortSolver {
    // ...
}
```
