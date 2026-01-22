# Type definitions

``` rust , ignore
type Option a = Some a | None;
type Pair a b = Pair a b;
type Result e a = Ok a | Err e;
```

- Variants can take **0 or more arguments**.
- **Newtype shorthand** is available when:
  - There is only one variant, and
  - The type name and constructor name are the same, and
  - The variant has exactly one tuple or one record argument.

``` rust , ignore
// Normal form
type Point a = Point @{ x: a, y: a };
type Wrapper a = Wrapper (a,);

// Newtype shorthand
type Point a = @{ x: a, y: a };
type Wrapper a = (a,);
```

