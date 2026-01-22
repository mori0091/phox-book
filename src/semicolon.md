# Semicolon (item separator)

- `;` separates multiple items (declarations / statements / expressions) in a block or at the top level.
- Each item is evaluated in order; only the last expression's value is returned.
- If a block or top-level input ends with `;`, an implicit `()` is added.

``` rust , ignore
// Multiple items in a block
{
    let x = 1; // => (): ()  (discarded)
    let y = 2; // => (): ()  (discarded)
    x + y;     // => 3: Int  (discarded)
    2 * x + y  // => 4: Int  (result)
}
// => 4: Int
```

``` rust , ignore
// Multiple items in the top level
let x = 1; // => (): ()  (discarded)
let y = 2; // => (): ()  (discarded)
x + y;     // => 3: Int  (discarded)
2 * x + y  // => 4: Int  (result)
// => 4: Int
```

``` rust , ignore
// Items in a block ends with `;`
{
    1 + 2; // => 3: Int  (discarded)
}
// => (): ()
```

``` rust , ignore
// Items in the top level ends with `;`
1 + 2; // => 3: Int  (discarded)
// => (): ()
```

``` rust , ignore
// No items in a block
{}
// => (): ()
```

