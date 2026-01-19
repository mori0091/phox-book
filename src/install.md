# Install Phox

> ⚠️ Work in progress — Phox is under active development.

# Build

Clone the GitHub repository and build with Rust (tested on 1.80+, may work on earlier versions):

```sh
git clone https://github.com/mori0091/phox.git
cd phox
cargo build
```

# Run in REPL

If you run without arguments, Phox starts an interactive REPL:

``` sh
cargo run
> let rec fact = λn. if (0 == n) 1 else n * fact (n - 1);
> fact 5
=> 120: Int
```

In REPL, an input starts with `:` is recognized as a REPL command.  
For example, `:?` shows the list of available commands:

``` sh
cargo run
> :?

:quit, :q
    exit REPL.

:help, :h, or :?
    print this help messages.

:load <path>, :l <path>
    load and evaluate Phox source file specified by <path>.

:modules, :m
    print list of root modules.

:using, :u
    print list of using aliases for each modules.

:symbols, :s
    print list of symbols for each modules.

:impls, or :impls [options]
    print list of `impl`s.
    options:
        -v, or --verbose
            print also the `impl`s' implementation.

```

# Run a program file

Pass a `.phx` file to execute it:

> - `.phx` is the conventional extension for Phox source files (plain text).  
> - `.txt` files are also accepted.

``` sh
cargo run examples/fact.phx
=> 120: Int
```

# Run from stdin

You can also pipe code from stdin (use `-` explicitly):

```sh
echo "1 + 2" | cargo run -
=> 3: Int
```

Resulting `value` and inferred `type` are printed on success, like this:

``` sh
=> value: type
```

On error, an error message is printed, like this:

``` sh
parse error: UnrecognizedToken { /* snip... */ }
```
