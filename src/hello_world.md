# First Program

```rust,ignore
> println "Hello World!"
Hello World!
=> Ok (): Result Error ()

> println ("Hello" ++ " " ++ "World!")
Hello World!
=> Ok (): Result Error ()

> println <| fmt::show <| fmt::sep_by "_" <| "Hello World!"
H_e_l_l_o_ _W_o_r_l_d_!
=> Ok (): Result Error ()

```

> [!NOTE]
> Phox is still under active development.  
> While most I/O primitives have not yet been implemented, but the following core features are already available:
> - Type-level Unicode String Framework (`ScalarString`, UTF-8, etc.)
> - Pretty-Printing Combinators
> - Rich pure-functional core libraries (iter, array, fmt, ...)
> - Experimental `print`, `println`, `eprint`, and `eprintln`
