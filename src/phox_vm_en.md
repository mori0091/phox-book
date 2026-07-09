# **STReAM: Suspendable Term Reduction Abstract Machine (overview)**

## 1. Introduction

We present **STReAM**, the *Suspendable Term Reduction Abstract Machine* designed for the strict functional language Phox.  
Although STReAM shares superficial similarities with classical machines such as the Krivine Machine, CEK, SECD, and STG, it diverges fundamentally through:

- strict (eager) evaluation,
- heap-first value representation,
- a two-layer structure of lexical vs. continuation environments,
- continuation sequencing via `CSeq`,
- a step-based reduction loop that flattens AST nodes,
- and a fully suspendable VM state.

STReAM is therefore a **new species of abstract machine**, not reducible to any existing model.

---

## 2. Machine State

STReAM decomposes its state into **three semantic scopes**:

- **lexical scope**    : the current term and its lexical environment  
- **contextual scope** : continuation code and continuation values  
- **global scope**     : global code table and heap

### 2.1 Formal State

> **VM state** = (`term`, `ctx.conts`, `ctx.env`, `g.codes`, `g.heap`)

```rust
State {
  term : Term,        // lexical scope
  ctx  : Context {    // continuation scope
    conts: CStack,
    env:   Env,
  },
  g : {               // global scope
    codes: GlobalEnv,
    heap:  Heap,
  },
}
```

> **Note**  
> `term` and `ctx` are specific to each VM instance,  
> while `g` may be shared across multiple VM instances.  
>  
> Thus, STReAM naturally supports **asynchronous and concurrent execution**  
> at the abstract machine level.

---

## 3. Instruction Set

### 3.1 Expressions

```
App(M, N)
CSeq(M, N)
LetRec(X, E)
Var(n)
GlobalVar(s)
For
Match(scrut, arms)
IndexAccess(t, i)
TupleAccess(t, n)
FieldAccess(t, label)
```

### 3.2 Value Constructors

```
Lit(L)
Tuple(n)
Con(name, n)
Record(labels)
Array(n)
ArrayU8(n)
...
```

### 3.3 Continuations

```
KApp
KMatch(arms)
KIndexAccess
KTupleAccess(n)
KFieldAccess(label)
KFor
KFor2
KLetRec(E)
```

---

## 4. Dynamics

STReAM’s `run_state()` always performs **exactly one AST-flattening step**,  
and repeated application of this step drives evaluation.

### 4.1 WHNF (termination)

If `ctx.conts` is empty and `term` is WHNF (`Lam` or `Val`), evaluation terminates.

### 4.2 WHNF with continuation

The current term is stored in the heap, its address is pushed to `ctx.env`,  
and the next continuation is loaded.

### 4.3 CSeq (Continuation Sequencer)

```
CSeq(M, N)
→ push continuation N (as closure)
→ replace current code with M
```

This separates “evaluate M” from “then execute N” at the AST level.

### 4.4 ACCESS / APP / LETREC

Variable lookup, strict application, and recursive binding follow the formal rules  
defined in the transition tables.

---

## 5. Comparison with Existing Machines

### 5.1 Krivine Machine

- shares closures and de Bruijn environments  
- differs in strict evaluation and flattened continuations

### 5.2 CEK Machine

- similar continuation stack  
- CEK continuations are recursive trees; STReAM continuations are flat sequences  
- CEK lacks the three-layer environment model

### 5.3 SECD Machine

- S/E/C/D correspond structurally to STReAM’s components  
- but STReAM uses AST + CSeq flattening instead of instruction lists

---

## 6. Suspend/Resume

Because STReAM’s state is fully linearized:

- **suspend** = copy the `State`  
- **resume** = restore the `State`

This enables:

- async/await  
- generators  
- coroutines  
- resumable pipelines  
- concurrent VM instances  

without modifying language semantics.

---

## 7. Conclusion

STReAM is a novel abstract machine characterized by:

- strict evaluation  
- heap-first value representation  
- AST-flattening semantics  
- continuation sequencing via CSeq  
- a three-layer environment model  
- suspendable VM state  

In summary:

> **“A strict, heap-first, flattening continuation machine with a three-layer environment model.”**

STReAM does not match CEK, SECD, STG, ZINC, or Krivine.  
It is a genuinely new architecture suited for modern language features such as async, pipelines, and pattern matching.
