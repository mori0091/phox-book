# [T.B.C.] Static Reference Analysis

## Overview

Static Reference Analysis tracks dependencies between Nodes through references.

Its primary purpose is to provide a common foundation for compiler analyses such as:

- Circular reference detection
- Alias analysis
- Reuse analysis
- Job-to-Job reference sharing analysis
- Escape and retention analysis

The analysis is based on a directed dependency graph of Nodes.

The central concepts are:

- `Roots`: the set of NodeIds directly referenced by a value or expression.
- `Deps`: the direct dependencies of a Node.
- `Reach`: the set of Nodes transitively reachable through those dependencies.
- `Store`: the mapping from NodeIds to their statically represented contents.

The analysis is performed after type inference and type checking.  
It does not evaluate expressions or inspect the runtime Heap or runtime addresses.

The analysis is intentionally conservative: if a dependency cannot be determined
statically, the analysis may report it as unknown rather than attempting to
evaluate the expression.

## 1. Runtime Model

The runtime memory model provides the basis for understanding the relationships
tracked by this analysis.

### 1.1 Runtime structures

The runtime Heap and environment are represented as:

```text
Heap := Addr -> Term
Env  := Vec<Addr>
```

A runtime term is one of:

```text
Term
    := Val(Value)
    | Clo(Code, Env)
```

A value is one of:

```text
Value
    := Lit(Lit)
    | Con { i1:Term, ..., iN:Term }
    | Ref(Addr)
```

A `Clo` contains the function code and the environment captured when the closure
was created.

A `Ref(a)` refers to the runtime address `a` in the Heap.

A `Con` contains its fields directly as Terms. Its fields do not have
independent runtime addresses merely by virtue of being fields.

### 1.2 Static and runtime representations

Static Reference Analysis does not use runtime `Addr` or inspect runtime `Heap`.

Instead, it introduces `NodeId` as a static identifier for a Node and `Store` as
a static representation of Node contents.

The correspondence is conceptual:

| Runtime          | Static analysis                                      |
|------------------|------------------------------------------------------|
| `Addr`           | `NodeId`                                             |
| `Heap`           | `Store`                                              |
| `Ref(Addr)`      | `Ref(NodeId)`                                        |
| `Clo(Code, Env)` | `Clo` with statically resolved captured dependencies |

A `NodeId` identifies a Node; it is not a runtime address.

## 2. Node and Store

### 2.1 NodeId

`NodeId` is a unique identifier assigned to each statically represented Node.

Its concrete representation is implementation-dependent.

### 2.2 Node

A Node is represented as:

\\[
Node(e,E)
\\]

where:

- \\(e\\) is the expression stored in the Node.
- \\(E\\) is the environment used to resolve the variables occurring in \\(e\\).

The expression is not evaluated. The Node retains its expression and environment
as static analysis information.

The identity of a Node (`NodeId`) is independent of its contents.

### 2.3 Store

The static Store maps NodeIds to Nodes:

\\[
Store : NodeId \rightarrow Node
\\]

Updating a Node changes the contents associated with its NodeId, but does not
change its identity.

For example:

\\[
Store' = Store[n \mapsto Node(e,E)]
\\]

replaces the contents of Node \\(n\\).

## 3. Roots

`Roots` describes the set of NodeIds directly referenced by a value.

\\[
Roots : Value \rightarrow \mathcal{P}(NodeId)
\\]

### 3.1 Literals

A literal does not reference any Node:

\\[
Roots(Lit(l))=\varnothing
\\]

### 3.2 References

A reference directly refers to its target Node:

\\[
Roots(Ref(n))=\\{n\\}
\\]

### 3.3 Constructors

For a constructor value:

\\[
Roots(Con(v_1,\ldots,v_k)) = \bigcup_{i=1}^{k}Roots(v_i)
\\]

Only direct references contained in its fields are included.

### 3.4 Closures

A closure may retain references through its captured environment.

For a closure:

\\[
Clo(\lambda x.e,E)
\\]

its roots are determined by the free variables of the function:

\\[
Roots(Clo(\lambda x.e,E)) = \bigcup_{v\in FV(\lambda x.e)}Roots(E(v))
\\]

where:

\\[
FV(\lambda x.e)=FV(e)\setminus\\{x\\}
\\]

The function body is not evaluated. Only its free variables and their captured
values are considered.

## 4. Roots_E

`Roots_E(e)` describes the set of NodeIds directly referenced by expression
\\(e\\) under environment \\(E\\).

\\[
Roots_E : Expr \times Environment \rightarrow \mathcal{P}(NodeId)
\\]

Unlike `Reach`, `Roots_E` does not recursively traverse the Store.

### 4.1 Variables

A variable refers to the value bound to it in the environment:

\\[
Roots_E(x)=Roots(E(x))
\\]

### 4.2 Literals

A literal has no direct references:

\\[
Roots_E(Lit(l))=\varnothing
\\]

### 4.3 Constructors

For a constructor expression:

\\[
Roots_E(Con(e_1,\ldots,e_k)) = \bigcup_{i=1}^{k}Roots_E(e_i)
\\]

The resulting constructor value contains the values of its fields, so its direct
references are the union of the references in those fields.

### 4.4 Function expressions

Evaluating a lambda expression creates a closure that captures its free
variables.

Therefore:

\\[
Roots_E(\lambda x.e) = Roots(Clo(\lambda x.e,E))
\\]

Equivalently:

\\[
Roots_E(\lambda x.e) = \bigcup_{v\in FV(\lambda x.e)}Roots(E(v))
\\]

The function body is not evaluated when calculating its roots.

### 4.5 Function application

For a function application:

\\[
Roots_E(e_1\ e_2) = Roots_E(e_1)\cup Roots_E(e_2)
\\]

Both the function expression and the argument expression may contribute
references to the resulting value.

This rule is conservative: it includes direct dependencies from both expressions
without evaluating the application.

### 4.6 Let expressions

A let expression:

\\[
let\ x=e_1\ in\ e_2
\\]

is equivalent to function application:

\\[
(\lambda x.e_2)\ e_1
\\]

Therefore, no separate `Roots_E` rule is required.

Its roots are determined by the function application and lambda rules.

### 4.7 Conditional expressions

For a conditional expression:

\\[
if\ e_1\ then\ e_2\ else\ e_3
\\]

the roots include the condition and both possible result expressions:

\\[
Roots_E(if\ e_1\ then\ e_2\ else\ e_3) = Roots_E(e_1) \cup Roots_E(e_2) \cup Roots_E(e_3)
\\]

The condition is included because it is evaluated as part of evaluating the
conditional expression.

Both branches are included because either branch may produce the result.

### 4.8 Match expressions

A match expression is handled in the same way as a conditional expression.

Its roots include:

- The scrutinee expression.
- Every possible branch expression.

Pattern-bound variables are resolved in the corresponding branch environment.

### 4.9 Direct dependencies only

Consider:

```phox
let x = new 10;
let y = new (Some x);
let z = Some y;
```

Suppose the corresponding NodeIds are \\(n_x\\) and \\(n_y\\).

The direct roots of the values are:

\\[
Roots(Ref(n_x))=\\{n_x\\}
\\]

\\[
Roots(Ref(n_y))=\\{n_y\\}
\\]

The expression `Some y` has:

\\[
Roots_E(Some\ y)=\\{n_y\\}
\\]

It does not directly contain \\(n_x\\).

The dependency on \\(n_x\\) is obtained by following the dependency of \\(n_y\\).  
This is the responsibility of `Reach`.

## 5. Expression Semantics

This section describes the meaning of each expression form relevant to Static
Reference Analysis.

The runtime semantics below describe how expressions affect the runtime Heap and
environment. The static rules describe how their reference dependencies are
represented.

### 5.1 Variable

```phox
x
```

A variable evaluates to the Term bound to it in the current environment.

Its static roots are:

\\[
Roots_E(x)=Roots(E(x))
\\]

### 5.2 Lambda expression

```phox
λx.e
```

A lambda expression evaluates to a closure containing:

- The function code \\(\lambda x.e\\).
- The environment captured at closure creation.

Its static roots are the roots of the captured free variables:

\\[
Roots_E(\lambda x.e)=\bigcup_{v\in FV(\lambda x.e)}Roots(E(v))
\\]

The body is not evaluated at closure creation.

### 5.3 Function application

```phox
e_1 e_2
```

A function application evaluates the function expression and argument
expression, then applies the resulting function to the argument.

For a closure, the function body is evaluated in the captured environment
extended with the argument binding.

Its static roots are:

\\[
Roots_E(e_1\ e_2)=Roots_E(e_1)\cup Roots_E(e_2)
\\]

### 5.4 Let expression

```phox
let x = e_1 in e_2
```

A let expression evaluates \\(e_1\\), binds the resulting value to \\(x\\), and
evaluates \\(e_2\\) in the extended environment.

It is equivalent to:

```phox
(λx.e_2) e_1
```

Consequently, its static dependencies are determined by the rules for lambda
expressions and function application.

### 5.5 Conditional expression

```phox
if e_1 then e_2 else e_3
```

A conditional evaluates its condition \\(e_1\\), then evaluates the selected branch.

Its static roots include all three expressions:

\\[
Roots_E(if\ e_1\ then\ e_2\ else\ e_3)=Roots_E(e_1)\cup Roots_E(e_2)\cup Roots_E(e_3)
\\]

The analysis does not attempt to determine which branch will execute.

### 5.6 Literal expression

```phox
10
```

A literal evaluates to a literal value:

\\[
Roots_E(Lit(l))=\varnothing
\\]

### 5.7 Constructor expression

```phox
Con(e_1, ..., e_k)
```

A constructor expression evaluates its field expressions and constructs a value
containing the resulting Terms.

Its static roots are:

\\[
Roots_E(Con(e_1,\ldots,e_k))=\bigcup_{i=1}^{k}Roots_E(e_i)
\\]

### 5.8 Allocation expression

```phox
new e
```

An allocation expression evaluates \\(e\\), allocates a new runtime address,
stores the resulting Term at that address, and returns a reference to it.

Conceptually:

\\[
\frac{
\begin{aligned}
&\langle e,E,Heap\rangle
\Downarrow
\langle t,Heap_1\rangle\\
&a\ \text{fresh}\\
&\\
&Heap_2=Heap_1[a\mapsto t]
\end{aligned}
}{
\langle new\ e,E,Heap\rangle
\Downarrow
\langle Ref(a),Heap_2\rangle
}
\\]

For static analysis, allocation creates a fresh NodeId \\(n\\) and adds a Node
to the Store:

\\[
\frac{
n\ \text{fresh}
\qquad
Store'=Store[n\mapsto Node(e,E)]
}{
\langle new\ e,E,Store\rangle
\longrightarrow
\langle Ref(n),Store'\rangle
}
\\]

Therefore:

\\[
Roots_E(new\ e)=\\{n\\}
\\]

The references contained in \\(e\\) are not directly included in the roots of
the allocation expression. They become dependencies of the newly created Node
and are obtained through `Deps` and `Reach`.

Allocation itself does not introduce a dependency from an existing Node to the
new Node. Since \\(n\\) is fresh, no existing Node can already depend on it.
Therefore, allocation does not require a circular reference check.

### 5.9 Reference assignment

```phox
r <- e
```

A reference assignment evaluates the left-hand side to identify the target
reference, evaluates the right-hand side, and replaces the contents stored at
the target address.

If the left-hand side resolves to `Ref(a)`, the runtime Heap is updated:

\\[
Heap'=Heap[a\mapsto t]
\\]

where \\(t\\) is the evaluated right-hand side.

For static analysis, if the left-hand side resolves to `Ref(n)`, the NodeId
remains unchanged. Only its contents in the Store are replaced:

\\[
Store'=Store[n\mapsto Node(e,E)]
\\]

The assignment is permitted only if it does not introduce a circular dependency.
The precise condition is defined in Section 8.

The right-hand side is represented by its expression and environment in the
updated Node. It is not evaluated by Static Reference Analysis.

## 6. Node Dependencies

The direct dependencies of a Node are determined by the roots of its contents.

First, define the roots of a Node:

\\[
Roots(Node(e,E)):=Roots_E(e)
\\]

Then define the direct dependencies of Node \\(n\\):

\\[
Deps_{Store}(n):=Roots(Store(n))
\\]

Therefore, if:

\\[
Store(n)=Node(e,E)
\\]

then:

\\[
Deps_{Store}(n)=Roots_E(e)
\\]

A directed edge:

\\[
n\rightarrow m
\\]

exists exactly when:

\\[
m\in Deps_{Store}(n)
\\]

This edge means that the contents of Node \\(n\\) directly reference Node \\(m\\).

For example:

```phox
let x = new Nil;
let y = new (Cons 10 x);
let z = new (Cons 20 y);
```

Let the corresponding NodeIds be \\(n_x,n_y,n_z\\).

The Store contains:

\\[
Store(n_x)=Node(Nil,E)
\\]

\\[
Store(n_y)=Node(Cons(10,x),E)
\\]

\\[
Store(n_z)=Node(Cons(20,y),E)
\\]

The direct dependencies are:

\\[
Deps_{Store}(n_x)=\varnothing
\\]

\\[
Deps_{Store}(n_y)=\\{n_x\\}
\\]

\\[
Deps_{Store}(n_z)=\\{n_y\\}
\\]

Thus, the dependency graph is:

```text
n_z -> n_y -> n_x
```

## 7. Reach

`Reach` represents transitive reachability through the Node dependency graph.

For a NodeId \\(n\\), define:

\\[
Reach(Store,n) = \\{n\\} \cup \bigcup_{m\in Deps_{Store}(n)} Reach(Store,m)
\\]

For a set of NodeIds \\(S\\), define:

\\[
Reach(Store,S) = \bigcup_{n\in S}Reach(Store,n)
\\]

Unlike `Roots`, `Reach` includes the starting Node itself.

For the previous example:

\\[
Reach(Store,n_x)=\\{n_x\\}
\\]

\\[
Reach(Store,n_y)=\\{n_y,n_x\\}
\\]

\\[
Reach(Store,n_z)=\\{n_z,n_y,n_x\\}
\\]

Thus:

- `Roots_E` describes direct references from an expression.
- `Deps` describes direct references from a Node.
- `Reach` describes all Nodes reachable through those references.

The Store is maintained as an acyclic graph. Therefore, the recursive definition
of `Reach` does not need to handle existing cycles.

## 8. Circular Reference Detection

Circular references are prevented by checking assignments that modify existing
Nodes.

### 8.1 Assignment condition

Suppose the left-hand side of an assignment resolves to:

\\[
Ref(n)
\\]

and the right-hand side is \\(e\\), evaluated in environment \\(E\\).

The assignment would replace the contents of Node \\(n\\) with:

\\[
Node(e,E)
\\]

The direct dependencies introduced by this replacement are:

\\[
Roots_E(e)
\\]

A circular dependency is introduced exactly when one of these dependencies can
already reach \\(n\\) through the current Store.

Therefore, the assignment is permitted if and only if:

\\[
\boxed{
n\notin Reach(Store,Roots_E(e))
}
\\]

If this condition holds, update the Store:

\\[
Store'=Store[n\mapsto Node(e,E)]
\\]

Otherwise, reject the assignment.

The check is performed against the Store **before** the assignment.

### 8.2 Example

Consider:

```phox
let x = new Nil;
let y = new (Cons 10 x);
let z = new (Cons 20 y);

x <- Cons 30 z;
```

Assume the NodeIds are \\(n_1,n_2,n_3\\), respectively.

Before the assignment:

\\[
Store(n_1)=Node(Nil,E)
\\]

\\[
Store(n_2)=Node(Cons(10,x),E)
\\]

\\[
Store(n_3)=Node(Cons(20,y),E)
\\]

The existing dependency graph is:

```text
n_3 -> n_2 -> n_1
```

The assignment:

```phox
x <- Cons 30 z
```

would replace the contents of \\(n_1\\) with:

\\[
Node(Cons(30,z),E)
\\]

The right-hand side has:

\\[
Roots_E(Cons(30,z))=\\{n_3\\}
\\]

Following the existing dependencies:

\\[
Reach(Store,\\{n_3\\})=\\{n_3,n_2,n_1\\}
\\]

Therefore:

\\[
n_1\in Reach(Store,\\{n_3\\})
\\]

The assignment is rejected because it would introduce the cycle:

```text
n_1 -> n_3 -> n_2 -> n_1
```

### 8.3 Acyclicity invariant

Assume that the initial Store is acyclic:

\\[
Store_0\text{ is acyclic}
\\]

Allocation creates a fresh Node and cannot introduce an edge from an existing
Node to that new Node.

Every assignment is checked using the condition in Section 8.1.

Replacing the contents of Node \\(n\\) introduces a cycle if and only if one of
the new direct dependencies can already reach \\(n\\).

The assignment condition rejects exactly this case.

Consequently, by induction, every permitted operation preserves the acyclicity
of the Store:

\\[
\boxed{
\forall t,\quad Store_t\text{ is acyclic}
}
\\]

This guarantee assumes that every operation capable of introducing or changing
Node dependencies is subject to the corresponding check.

## 9. Applications

The dependency graph provides a common foundation for further compiler analyses.

### 9.1 Alias analysis

The Nodes reachable from an expression can be obtained from its roots:

\\[
NodeReach_E(e)=Reach(Store,Roots_E(e))
\\]

If two expressions have intersecting reachable Node sets:

\\[
NodeReach_E(e_1)\cap NodeReach_E(e_2)\neq\varnothing
\\]

they may refer to a common Node.

This provides a basis for may-alias analysis.

More precise may-alias and must-alias analyses can be built on top of this
information.

### 9.2 Reuse analysis

To determine whether a Node can be reused for an in-place update, the compiler
needs to know whether another live value can observe its contents.

The dependency graph provides the reference and alias information required for
this analysis.

Combined with liveness and mutation information, it can help distinguish between:

- Nodes reachable only from the value being updated.
- Nodes that may also be reachable from other live values.

The latter cannot be modified in place without considering the effects on those
other values.

### 9.3 Job-to-Job reference sharing analysis

For concurrent execution, the Nodes reachable from values captured or retained
by each Job can be calculated.

For two Jobs \\(A\\) and \\(B\\), define:

\\[
SharedNodes(A,B) = NodeReach_E(A)\cap NodeReach_E(B)
\\]

If this set is non-empty, the Jobs may share Nodes.

Combining this information with access modes, such as read-only and mutable
access, can help determine whether synchronization is necessary.

The analysis can be extended to include values retained by Channels and other
runtime structures.

### 9.4 Escape and retention analysis

The same reachability information can be used to determine whether a Node
escapes a particular scope or Job, and which values may retain references to it.

This can provide useful information for memory management and lifetime analysis.

## 10. Compiler Pipeline

Static Reference Analysis is performed after successful type inference and type
checking.

A possible compiler pipeline is:

1. Parse source code.
2. Perform type inference and type checking.
3. Construct the static environments and initial Store.
4. Calculate and cache `Roots_E` for expressions.
5. Calculate `Deps` for Nodes.
6. Construct or query the dependency graph.
7. Validate assignments using the circular reference condition.
8. Perform further analyses such as alias analysis, reuse analysis, and Job-to-Job sharing analysis.

The analysis is independent of the Hindley–Milner type system and type-class
resolution, except that it assumes expressions have already passed type
checking.

