# Comments

## Line comments
> In regular expression: `//[^\n\r]*`

- starts with `//`, followed by any characters (single-line text), and ends with newline.

```
// This is a line-comment.

////// This is also a line-comment.
```

> Similar to C/C++ line-comments.

## Block comments
> In regular expression: `/\*[^*]*\*+(?:[^/*][^*]*\*+)*/`

- starts with `/*`, followed by any charcters (single or multi-line texts), and ends with `*/`.
- Nesting is not allowed.

``` rust , ignore
/* This is a block comment. */

/*
 * This is also a block comment.
 */

/*****
 *** This is also a block comment.
 **/
```

> Similar to C/C++ block-comments.
