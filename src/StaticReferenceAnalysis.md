# [T.B.C.] Static Reference Analysis

## Overview

This document describes a static analysis framework for tracking dependencies between values and references.

The primary purpose is to establish a common foundation for several compiler analyses:

- Circular reference detection
- Alias analysis
- Reuse analysis
- Job-to-Job reference sharing analysis
- Escape and retention analysis

The analysis is based on a dependency graph constructed from the relationships between variables and the values they represent.

The central concept is `Roots`, which describes the set of variables that the evaluation result of an expression directly depends on. Transitive dependencies are represented by `Reach`, which is derived from the dependency graph.

The analysis is intended to be performed after type inference and type checking. It is independent of the Hindley–Milner type system and type-class resolution.

## 1. Value Categories and Environment

We distinguish three categories of values.

### 1.1 Leaf

A Leaf is an ordinary value, or a composite value consisting only of Leaf values.

A Leaf contains no references to Nodes.

Examples include:

- Unit, Boolean, and numeric values
- Tuples and records consisting only of Leaf values
- Algebraic data type values consisting only of Leaf values

### 1.2 Node

A Node is a composite value that may contain references to other Nodes.

Each Node has its own identity, represented by a variable.

A Node may contain:

- Leaf values
- References to Nodes

### 1.3 Ref

A Ref is a reference to a Node.

`Ref(x)` denotes a reference to the Node represented by variable `x`.

### 1.4 Environment

An environment maps variables to their defining expressions.

\\[
E : Variable \\rightharpoonup Expr
\\]

The set of variables representing Nodes is:

\\[
N_E = \\{x \\in dom(E) \\mid E(x) \\text{ evaluates to a Node}\\}
\\]

Variables may be bound to Leaf, Node, or Ref values.

The classification of expressions and variables is determined after type inference and type checking.

## 2. Roots

We define `Roots(e)` as the set of variables that the evaluation result of expression `e` directly depends on.

\\[
Roots : Expr \\rightarrow \\mathcal{P}(Variable)
\\]

The definition is syntax-directed. It describes direct dependencies only and does not recursively expand variable bindings through the environment.

### 2.1 Basic expressions

For constants:

\\[
Roots(c) = \\varnothing
\\]

For variables:

\\[
Roots(x) = \\{x\\}
\\]

For references:

\\[
Roots(Ref(x)) = \\{x\\}
\\]

### 2.2 Composite values

For a composite value consisting of expressions \\(e_1, \\ldots, e_n\\):

\\[
Roots(Node(e_1, \\ldots, e_n)) = \\bigcup_{i=1}^{n} Roots(e_i)
\\]

The same principle applies to other composite values.

A composite value consisting only of Leaf values has an empty root set.

### 2.3 Conditional expressions

For a conditional expression, only the possible result expressions contribute to `Roots`.

\\[
Roots(if\\ c\\ then\\ e_1\\ else\\ e_2) = Roots(e_1) \\cup Roots(e_2)
\\]

The condition `c` is excluded.

For example:

```text
z = if (x > y) a else b
```

The result may depend on either `a` or `b`, but does not retain `x` or `y` merely because they were used to evaluate the condition.

Therefore:

\\[
Roots(E(z)) = Roots(a) \\cup Roots(b)
\\]

### 2.4 Function expressions and closures

For a function expression:

\\[
Roots(\\lambda x.e) = FV(e) \\setminus \\{x\\}
\\]

Here, `FV(e)` denotes the free variables of `e`.

The free variables are captured by the closure and may therefore contribute to its dependencies.

> [!NOTE]
> `Roots` tracks dependencies of the evaluation result, not every variable accessed during evaluation. Dependencies used only to evaluate a conditional or another intermediate computation are not included. If temporary lifetimes or concurrent accesses during evaluation need to be analyzed, they should be handled by a separate analysis.

### 2.5 Direct dependencies only

Consider the following example:

```text
x = Node(...)
y = Ref(x)
z = Node(first = y, second = 20)
```

The direct roots are:

\\[
Roots(E(x)) = \\varnothing
\\]

\\[
Roots(E(y)) = \\{x\\}
\\]

\\[
Roots(E(z)) = \\{y\\}
\\]

In particular, `Roots(E(z))` is `{y}`, not `{x, y}`.

The dependency on `x` is obtained by following the dependency of `y`. This is the responsibility of `Reach`.

`Roots` can be computed and cached independently for each expression.

## 3. Dependency Graph

Using `Roots`, we construct a directed dependency graph for an environment.

\\[
G_E = (V_E, A_E)
\\]

where:

\\[
V_E = dom(E)
\\]

\\[
A_E = \\{(x,y) \\mid x \\in V_E \\land y \\in Roots(E(x))\\}
\\]

An edge \\(x \\rightarrow y\\) means that the value defined by `x` directly depends on the value represented by `y`.

The graph includes variables bound to Leaf, Node, and Ref values. Paths may therefore pass through variables bound to Ref values.

For the previous example, the graph contains:

```text
z -> y -> x
```

This representation distinguishes direct dependencies from transitive dependencies.

## 4. Reach

`Reach` represents transitive reachability in the dependency graph.

First, define the direct dependencies of a variable:

\\[
Deps_E(x) = Roots(E(x))
\\]

Then define reachability recursively:

\\[
Reach_E^{(1)}(x) = Deps_E(x)
\\]

\\[
Reach_E^{(k+1)}(x) = \\bigcup_{y \\in Reach_E^{(k)}(x)} Deps_E(y)
\\]

The complete reachable set is:

\\[
Reach_E(x) = \\bigcup_{k=1}^{\\infty} Reach_E^{(k)}(x)
\\]

\\(Reach_E(x)\\) contains all variables reachable from `x` by following one or more dependency edges.

It is not reflexive: `x` is not included unless a cycle leads back to `x`.

For the example:

```text
x = Node(...)
y = Ref(x)
z = Node(first = y, second = 20)
```

we obtain:

\\[
Reach_E(x) = \\varnothing
\\]

\\[
Reach_E(y) = \\{x\\}
\\]

\\[
Reach_E(z) = \\{y,x\\}
\\]

Thus, `Roots` describes direct dependencies, while `Reach` describes transitive dependencies.

## 5. Safe Reference Assignment

Consider an assignment that stores a reference into a Node:

```text
n[i] = Ref(r)
```

This introduces a dependency edge:

```text
n -> r
```

To preserve an acyclic dependency graph, the assignment must not introduce a path from `r` back to `n`.

Therefore, the assignment is safe if:

\\[
n \\notin Reach_E(r)
\\]

We define the corresponding constraint:

\\[
AssignSafe(n[i] \\leftarrow Ref(r)) \\iff n \\notin Reach_E(r)
\\]

### 5.1 Replacing an existing reference

When an existing reference is overwritten, its old dependency edge must be removed before checking the new assignment.

Otherwise, a valid replacement might be incorrectly rejected because the old edge still contributes to the reachability relation.

Let \\(E^-\\) denote the environment or dependency graph after removing the old reference edge(s).

The correct condition is:

\\[
AssignSafe(n[i] \\leftarrow Ref(r)) \\iff n \\notin Reach_{E^-}(r)
\\]

The new edge \\(n \\rightarrow r\\) is added only after this condition has been verified.

## 6. Acyclicity Invariant

Assume that the initial dependency graph is acyclic:

\\[
G_{E_0} \\text{ is a DAG}
\\]

Every subsequent reference assignment is required to satisfy `AssignSafe`.

Adding an edge \\(n \\rightarrow r\\) creates a cycle if and only if a path from `r` to `n` already exists.

Since `AssignSafe` rejects exactly this case, every permitted assignment preserves acyclicity.

Consequently, by induction:

\\[
\\forall t,\\quad G_{E_t} \\text{ is a DAG}
\\]

This gives a static guarantee that the dependency graph contains no cycles, provided that all operations capable of introducing reference edges are subject to this constraint.

## 7. Cached Analysis Information

The analysis can cache the following information:

| Information      | Definition       | Purpose                              |
|------------------|------------------|--------------------------------------|
| Roots            | \\(Roots(e)\\)   | Direct syntactic dependencies        |
| Deps             | \\(Deps_E(x)\\)  | Direct dependencies of a variable    |
| Reach            | \\(Reach_E(x)\\) | Transitive dependencies              |
| Dependency graph | \\(G_E\\)        | Graph representation of dependencies |

`Roots` is computed per expression and can be reused.

`Reach` can be computed on demand, particularly when validating reference assignments. Depending on the implementation, it may also be cached and incrementally updated.

The distinction is important:

- `Roots` is a local, syntax-directed property of an expression.
- `Reach` is a transitive property derived from the dependency graph.
- The dependency graph provides a common representation that can be reused by multiple analyses.

## 8. Applications

The dependency graph provides a common foundation for several analyses.

### 8.1 Circular reference detection

A cycle in the dependency graph indicates a circular dependency.

By enforcing `AssignSafe` on every reference assignment, the compiler can guarantee that such cycles are never created.

### 8.2 Alias analysis

We can obtain the Nodes reachable from a variable by filtering its reachable set:

\\[
NodeReach_E(x) = Reach_E(x) \\cap N_E
\\]

For an expression `e`, the reachable Nodes can be obtained from its direct roots:

\\[
NodeReach_E(e) = \\bigcup_{x \\in Roots(e)} NodeReach_E(x)
\\]

If two values have intersecting reachable Node sets, they may refer to a common Node:

\\[
NodeReach_E(a) \\cap NodeReach_E(b) \\neq \\varnothing
\\]

This provides a basis for alias analysis.

More precise may-alias and must-alias analyses can be built on top of this information.

### 8.3 Reuse analysis

When considering whether a Node can be reused for an in-place update, we need to determine whether another live value can observe its current contents.

The dependency graph provides the reference and alias information needed for this analysis.

Combined with liveness analysis and mutation information, it can help determine whether a Node is safe to reuse.

In particular, the analysis needs to distinguish between:

- Nodes reachable only from the value being updated.
- Nodes that may also be reachable from other live values.

The latter cannot be modified in place without considering the effects on those other values.

### 8.4 Job-to-Job sharing analysis

For concurrent execution, we can calculate the Nodes reachable from values captured or retained by each Job.

For two Jobs, `A` and `B`, define:

\\[
SharedNodes(A,B) = NodeReach_E(A) \\cap NodeReach_E(B)
\\]

If this set is non-empty, the Jobs may share Nodes.

Combining this information with access modes (read-only or mutable) can help determine whether synchronization is necessary.

This analysis can be extended to account for values retained by Channels and other runtime structures.

### 8.5 Escape and retention analysis

The same reachability information can also be used to analyze whether a Node escapes a particular scope or Job, and which values may keep it alive.

This can provide useful information for memory management and lifetime analysis.

## 9. Compiler Pipeline

The analysis is independent of type inference and type-class resolution.

A possible compiler pipeline is:

1. Parse source code.
2. Perform type inference and type checking.
3. Classify expressions and variables for reference analysis.
4. Calculate and cache `Roots`.
5. Construct the dependency graph.
6. Generate and validate `AssignSafe` constraints.
7. Perform further analyses such as alias analysis, reuse analysis, and Job-to-Job sharing analysis.

By running reference analysis only after successful type checking, it can remain largely orthogonal to the Hindley–Milner type system and MPTC resolution.

