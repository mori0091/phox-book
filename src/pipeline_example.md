# Pipeline Example

``` rust,ignore
> counter 1 |> take 5 |> fold (+) 0
=> 15: Int

> counter 1 |> take 10 |> fold (+) 0
=> 55: Int

> counter 1 |> take 5 |> fold (*) 1
=> 120: Int

> counter 1 |> map #(* 2) |> take 5 |> fold (flip Cons) Nil
=> Cons 10 (Cons 8 (Cons 6 (Cons 4 (Cons 2 Nil)))): List Int

> counter 1 |> filter (\x. x / 2 * 2 == x) |> take 5 |> collect Nil
=> Cons 2 (Cons 4 (Cons 6 (Cons 8 (Cons 10 Nil)))): List Int

```
