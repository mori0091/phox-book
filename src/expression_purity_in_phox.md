# 🍀 Expression Purity in Phox

Although “pure” / "purity" is defined differently across programming languages,  
what purity ultimately seeks is clear:  
to identify expressions whose meaning does not depend on side-effects,  
and can therefore be safely evaluated statically.  

The following explains “expression purity” in Phox.

---

## Pure values
All values in Phox are pure and referentially transparent.

## Pure expressions
Most expressions are also pure.  
Any expression that does **not** contain a procedure is pure.

## Impure expressions
Expressions that contain a procedure — either procedural block (`proc!{...}`)  
or procedure call (calling a procedure such as `foo!()`) — are impure.  
Expressions that contain a procedure implicitly in nested sub-expressions are also impure.  
Impure expressions cannot be statically evaluated.

> [!NOTE]
> The “procedure” language feature is planned to be provided
> by the *proc system* in future developments.

## Static evaluation
Any pure value or pure expression **can be reduced to a constant**.  
The compiler is free to evaluate pure expressions at compile-time  
and embed their results as constant values,  
depending on code-size and compile-time cost heuristics.

> Pure expressions are referentially transparent,  
> so compile-time evaluation never changes program semantics.
