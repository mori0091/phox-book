# Pipeline Example

## *Iterator / Generator / Sink* pipelines

``` rust , ignore
> counter 1 |> take 5 |> fold (+) 0
=> 15: Int

> counter 1 |> take 10 |> fold (+) 0
=> 55: Int

> counter 1 |> take 5 |> fold (*) 1
=> 120: Int

> counter 1 |> filter (|(% 2) >> |(== 0)) |> take 5 |> collect Nil
=> Cons 2 (Cons 4 (Cons 6 (Cons 8 (Cons 10 Nil)))): List Int

> counter 1 |> filter (|(% 2) >> |(== 0)) |> take 5 |> collect @[]
=> @[2, 4, 6, 8, 10]: @[Int]

> collect "abc" "def"
=> "abcdef": ScalarString

> "abc" ++ "def"
=> "abcdef": ScalarString

> @[1,2,3] ++ @[10,20]
=> @[1, 2, 3, 10, 20]: @[Int]

> "Hello" ++ " " ++ "World"
=> "Hello World": ScalarString

```

> [!NOTE]  
> The infix operator `++` is synonym of `collect` function.


## *Pretty-Printing combinator* pipelines

``` rust , ignore
> use ::core::fmt::*;
=> (): ()

> show <| hex-dump <| cast "🎉🤣👍🍺"
=> "F0 9F 8E 89 F0 9F A4 A3 F0 9F 91 8D F0 9F 8D BA": ScalarString

> show <| sep_by ", " <| map single-quote <| "🎉🤣👍🍺"
=> "'🎉', '🤣', '👍', '🍺'": ScalarString

```

> [!NOTE]  
> The `::core::fmt` module is **not imported by default**.  
> You must either:
> - call functions with a module path such as `fmt::show`, or  
> - explicitly import the module with `use ::core::fmt::*;`
