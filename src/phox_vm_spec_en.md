# STReAM: Suspendable Term Reduction Abstract Machine (formal semantics)

- **STReAM** is *Suspendable Term Reduction Abstract Machine* designed for  
  strict functional programming languages.  
- **Phox VM** is an implementation of *STReAM* designed for  
  the *Phox* programming language.
- **Phox** is a strict functional programming language.

---

## Machine State

STReAM decomposes its state into **three semantic scopes**:

- **lexical scope**    : the current term and its lexical environment  
- **contextual scope** : continuation code and continuation values  
- **global scope**     : global code table and heap

### Formal State

> **VM state** = (`term`, `ctx.conts`, `ctx.env`, `g.codes`, `g.heap`)

```rust , ignore
State {
  // lexical scope
  term : enum Term {
    Val(Value),       // a value, or
    Clo(Closure {     // a closure
      code: Code,     // - code of the closure
      env: Env,       // - variables bounded to the closure
    }),
  },
  // contextual scope
  ctx : Context {
    conts: CStack,    // continuation closure-stack
    env: Env,         // continuation value-stack
  },
  // global scope
  g : {
    codes: GlobalEnv, // global code table
    heap: Heap,       // global store
  },
}
```

> [!Note]
> `term` and `ctx` are *specific to each VM instance*, but  
> `g` is *sharable among multiple VM instances*.  
> 
> In other words,
> - By their very nature, one VM instance can represent *one suspendable tasks or jobs*, and
> - Multiple VM instances (i.e., multiple tasks/jobs) can run in concurrent,
>   if the global allocator `g.heap` is multithread-safe,   
> 
> Consequently, the STReAM/Phox VM can naturally support asynchronous and
> concurrent computation at the abstract machine and runtime system levels.

----

## Notations of VM state

- `term`
  : Term. A Term is Closure or Value.
  - `t` means an arbitrary Closure or Value.
  - `<val>` means an arbitrary Value.
    - `val`
      : Value
      - ...
  - `{code, env}` means a Closure. Closure is pair of Code `code` and Env `env`.
    - `code`
      : Code (Instruction)
      - `Lit L` | `Var n` | `Lam E` | `App M N` | ...
    - `env`
      : Environment stack (Env)
      - `[]` means empty Env.
      - `es` means an arbitrary Env.
      - `a::es` means an Env whose top is `a`, where `a` is an address of heap
      - `es[n ↦ a]` means Env `es` whose element at de Bruijn index `n` is address `a`

- `ctx.conts`
  : Continuation closure-stack (CStack)
  - `[]` means empty CStack
  - `ks` means an arbitrary CStack.
  - `k::ks` means a CStack whose top is `k`, where `k` is a Closure.

- `ctx.env`
  : continuation value-stack (WStack ≡ Env)
  - `[]` means empty WStack
  - `ws` means an arbitrary WStack.
  - `a::ws` means a WStack whose top is `a`, where `a` is an address of heap

- `g.codes`
  : Random access read-only code table. (GlobalEnv)
  - `gs` means an arbitrary GloalEnv.
  - `gs[s ↦ c]` means GloalEnv `gs` whose element at key `s` is code `c`

- `g.heap`
  : Random access heap memory (Heap)
  - `h` means an arbitrary Heap.
  - `h[a ↦ t]` means Heap `h` whose element at address `a` is term `t`
  - `h[a ↦ {}]` means heap `h` whose element at address `a` is nil  
    (i.e. `a` is fresh address to be allocated later)

> [!NOTE]
> `h[a ↦ {}]` does *not* allocate memory.  
> It only denotes that `a` is a fresh address.  
> Actual allocation occurs when a value is written to `a`.  

---

## Dynamics of VM state transition

- WHNF (end of state transition)
- WHNF w/ continuation
- CSEQ (Continuation Sequencer)
- ACCESS (variable lookup)
- APP (function application)
- LET (let binding)
- LETREC (recursive binding)


### WHNF (end of state transition)

Evaluation halts when the current term was `{Lam E, es}` or `<val>` and there is no continuations.

| (rule) | term          | ctx.conts | ctx.env | g.codes | g.heap |
|:-------|---------------|-----------|---------|---------|--------|
| (Done) | `{Lam E, es}` | `[]`      | `[]`    | `gs`    | `h`    |

| (rule) | term    | ctx.conts | ctx.env | g.codes | g.heap |
|:-------|---------|-----------|---------|---------|--------|
| (Done) | `<val>` | `[]`      | `[]`    | `gs`    | `h`    |


### WHNF w/ continuation

Save the current term to the heap, push its address to `ctx.env`, and load the next continuation.

- Allocate fresh address `a` of heap for the current term,
- Push `a` to `ctx.env`,
- Pop continuation from `ctx.conts`.

| (rule) | term          | ctx.conts | ctx.env | g.codes | g.heap               |
|:-------|---------------|-----------|---------|---------|----------------------|
| cont   | `{Lam E, es}` | `k::ks`   | `ws`    | `gs`    | `h[a ↦ {}]`          |
| →      | `k`           | `ks`      | `a::ws` | `gs`    | `h[a ↦ {Lam E, es}]` |

| (rule) | term    | ctx.conts | ctx.env | g.codes | g.heap         |
|:-------|---------|-----------|---------|---------|----------------|
| cont   | `<val>` | `k::ks`   | `ws`    | `gs`    | `h[a ↦ {}]`    |
| →      | `k`     | `ks`      | `a::ws` | `gs`    | `h[a ↦ <val>]` |


### CSEQ (Continuation Sequencer)

- `CSeq M N`
  : Evaluate `M`, and then `N`.  
    Since `N` is evaluated after `M`,  
    `N` is pushed onto the continuation stack as a closure, and  
    the current code is replaced with `M`.

- Push continuation code `N` (as closure `{N, es}`) to `ctx.conts`,
- Replace the current code with `M`.

| (rule) | term             | ctx.conts     | ctx.env | g.codes | g.heap |
|:-------|------------------|---------------|---------|---------|--------|
| cseq   | `{CSeq M N, es}` | `ks`          | `ws`    | `gs`    | `h`    |
| →      | `{M, es}`        | `{N, es}::ks` | `ws`    | `gs`    | `h`    |


### ACCESS (variable lookup)

- `Var n`
  : Load term of a variable bounded the current env.

| (rule) | term                 | ctx.conts | ctx.env | g.codes | g.heap     |
|:-------|----------------------|-----------|---------|---------|------------|
| access | `{Var n, es[n ↦ a]}` | `ks`      | `ws`    | `gs`    | `h[a ↦ t]` |
| →      | `t`                  | `ks`      | `ws`    | `gs`    | `h[a ↦ t]` |


If de Bruijn index `n` was out of bounds, causes run-time error "variable not found".

| (rule)  | term                  | ctx.conts | ctx.env | g.codes | g.heap |
|:--------|-----------------------|-----------|---------|---------|--------|
| (Error) | `{Var n, es[n ↦ {}]}` | `ks`      | `ws`    | `gs`    | `h`    |


### APP (function application)

- `App M N`
  : Evaluate `M` and `N` in order, then apply the resulting function to the
    argument via `KApp`.

| (rule) | term            | ctx.conts                 | ctx.env | g.codes | g.heap |
|:-------|-----------------|---------------------------|---------|---------|--------|
| app    | `{App M N, es}` | `ks`                      | `ws`    | `gs`    | `h`    |
| →      | `{N, es}`       | `{M, es}::{KApp, []}::ks` | `ws`    | `gs`    | `h`    |

| (rule) | term         | ctx.conts | ctx.env    | g.codes | g.heap                       |
|:-------|--------------|-----------|------------|---------|------------------------------|
| kapp   | `{KApp, []}` | `ks`      | `f::x::ws` | `gs`    | `h[f ↦ {Lam E, es}, x ↦ tN]` |
| →      | `{E, x::es}` | `ks`      | `ws`       | `gs`    | `h[x ↦ tN]`                  |

where:  
- `{Lam E, es}` = resulting term (WHNF) of `{M, es}` via `cont` transition
- `tN` = resulting term (WHNF) of `{N, es}` via `cont` transition


### LET (let binding)

The code `Let X E` is synonym of `App (Lam E) X`.


### LETREC (recursive binding)

- `LetRec X E`
  : Allocate a dummy for recursive binding, evaluate `X`, then update the dummy
    with the result and evaluate `E`.

| (rule) | term               | ctx.conts                | ctx.env | g.codes | g.heap         |
|:-------|--------------------|--------------------------|---------|---------|----------------|
| letrec | `{LetRec X E, es}` | `ks`                     | `ws`    | `gs`    | `h[f ↦ {}]`    |
| →      | `{X, f::es}`       | `{KLetRec E, f::es}::ks` | `ws`    | `gs`    | `h[f ↦ dummy]` |

| (rule)  | term                 | ctx.conts | ctx.env | g.codes | g.heap                 |
|:--------|----------------------|-----------|---------|---------|------------------------|
| kletrec | `{KLetRec E, f::es}` | `ks`      | `x::ws` | `gs`    | `h[x ↦ tX, f ↦ dummy]` |
| →       | `{E, f::es}`         | `ks`      | `ws`    | `gs`    | `h[f ↦ tX]`            |

where:  
- `dummy` = an arabitrary *allocated* dummy term.
- `f` = an address that  
  - holds `dummy` at first via `letrec` transition, and then  
  - be updated with term at `x` later via `kletrec` transition.
  - finally `f` holds `tX` (the recursive function body).
- `tX` = resulting term (WHNF) of `{X, f::es}` via `cont` transition

