# Language constructs

## Identifiers

### Variables and type variables
> In regular expression:`[a-z_]([-]*[a-zA-Z0-9_]+)*['?]*`
- `seq`, `x'`, `some?`, `foo-bar`
- `fooBar`, `a0`, `take_while`

### Constructor, type constructor, and trait names
> In regular expression:`[A-Z][a-zA-Z0-9_]*`
- `List`, `Nil`, `Eq`, `Iter`

### Module names
> In regular expression:`[a-z_]([-]*[a-zA-Z0-9_]+)*`
- `core`, `cmp`, `iter`

### Module path (qualified module names)
> A sequence of module names separated by `::`
- `::core::cmp`, `::core::list` Absolute path
- `cmp`, `list`, `foo::bar` Relative path

### Qualified names
> A module path followed by a variable, constructor, type constructor, or trait name
- `::core::iter::seq`, `::core::iter::Seq`, `::core::iter::Iter`
- `iter::seq`, `iter::Seq`, `iter::Iter`

## Literals
- `()` Unit value literal
- `true`, `false` Boolean value literals
- `0`,`100`, `-1` Integer value literals

## Primitive types
- `()` Unit type
- `Bool` Boolean type
- `Int` Integer type
- `(t,)`, `(t1, t2)` Tuple type
- `@{x:t1, y:t2}` Record type
- `t1 -> t2` Function type
- `T t`, `t1 t2` Type-level function application

## Operators

### Infix (binary) operators
- `==`, `!=` Equality / Non-equality operators
- `<=`, `<`, `>`, `>=` Comparison operators
- `&&`, `||` Logical operators
- `+`, `-`, `*`, `/` Arithmetic operators (TODO: add `%` modulo)
- `|>`, `<|` Pipeline operators
- `>>`, `<<` Function composition operators

### Prefix operators
- `nagete`, `-` Arithmetic negation
- `not`, `!` Boolean not

### User defined infix operators
> In regular expression:`[*+\-/!$%&=^?<>]+`
- `?>`, `===`, `++`

### Operators as functions
- `(==) 1 1`, `(+) 1 2`

### Functions and constructors as operators
- ``1 `Cons` Nil``, ``x `@{Eq Int}.(==)` y``

## Declarations
- `type` Define Algebraic Data Type (ADT)
- `trait` Define Multi-Parameter Type Class (MPTC)
- `impl` Define type class instance implementation
- `*let` Define generic function template

## Statements
- `mod` Define sub-module (TODO: move to declarations)
- `use` Import identifiers (TODO: support local scope)
- `let` Variable / function binding
- `let rec` Recursive function binding

## Expressions

### Value expressions
> (evaluate to themselves; safe to generalize)
- `()`, `true`, `0` Literals
- `λp.e`/`\p.e` Lambda abstraction (function)
- `(e1,)`, `(e1, e2)` Tuple value
- `@{x:e1, y:e2}` Record value
- `@{T t1 t2}` Trait record value
- `#(e op)`/`#(e op _)` Infix-operator partial application (bind the 1st argument)
- `#(op e)`/`#(_ op e)` Infix-operator partial application (bind the 2nd argument)

> Infix-operator partial applications (a.k.a. section syntax) are desugared into lambda abstraction.

### Non-value expressions
> (require evaluation; not eligible for generalization)
- `x` Evaluate variable
- `e1 e2` Function application
- `e.0` Tuple index access
- `e.x` Record field access
- `{ stmt1; stmt2; expr }` Block expression (evaluates to the last expression; introduces a new scope)
- `if (e1) e2 else e3` If then else
- `match (e) { p1 => e1, .. }` Pattern matching

## Patterns
- `_` Wildcard pattern
- `()`, `true`, `0` Literal pattern
- `x`, `foo` Variable pattern
- `Nil`, `Cons p ps` Constructor pattern
- `(p1,)`, `(p1, p2)` Tuple pattern
- `@{x:p1, y:p2}`, `@{x, y}` Record pattern
