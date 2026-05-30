# Primitive Types

| type           | values                                         | description                                    |
|----------------|------------------------------------------------|------------------------------------------------|
| `()`           | `()`                                           | Unit type, that consists of single value `()`. |
| `Bool`         | `true`, `fales`                                | Boolean type. True or false.                   |
| `Int`          | `0`, `100`, `-1`, ...                          | Integer type. (currently 64-bit integer)       |
| `u8`           | `0u8`, `100u8`, `0xFFu8`, ...                  | 8-bit unsigned integer type.                   |
| `u16`          | `0u16`, `100u16`, `0xFFFFu16`, ...             | 16-bit unsigned integer type.                  |
| `u32`          | `0u32`, `100u32`, `0xFFFFFFFFu32`, ...         | 32-bit unsigned integer type.                  |
| `u64`          | `0u64`, `100u64`, `0xFFFFFFFFFFFFFFFFu64`, ... | 64-bit unsigned integer type.                  |
| `ScalarValue`  | `'A'`, `'\n'`, `'🎉'`, ...                    | Unicode Scalar Value (i.e. character)          |
| `ScalarString` | `"abc"`, `"🎉🤣👍🍺"`, ...                  | Unicode UTF-8 string                           |

> [!NOTE]
> `ScalarString` is synonym of `Str ScalarValue` type.
