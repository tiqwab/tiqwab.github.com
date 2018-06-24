---
layout: post
title: Go の array と slice でハマった
tags: "array, slice, go"
comments: true
---

最近 Go を始めました。構文はそんなに難しく無さそうだし書きながら覚えられるかなーと雑に始めたのですが、array と slice の区別ができておらずいくつかドハマりしました。またけっこう馴染みのない挙動を示すこともあったのでそのあたりを書き残します。

最終的に array と slice の違いについては [Effective Go][1] を読むことで整理ができました。

[Effective Go の Arrays 項][2] からの引用:

> There are major differences between the ways arrays work in Go and C. In Go, 
>
> - Arrays are values. Assigning one array to another copies all the elements. 
> - In particular, if you pass an array to a function, it will receive a copy of the array, not a pointer to it. 
> - The size of an array is part of its type. The types [10]int and [20]int are distinct.

[Effective Go の Slices 項][3] からの引用:

> Slices hold references to an underlying array, and if you assign one slice to another, both refer to the same array.

### array のコピーは配列全体のコピー

上で引用した Effective Go にも書いてあることですが、一応コードでも確認しておきます。

```go
package main

import (
    "fmt"
)

func withArray1(arr [1]int) {
    arr[0] = arr[0] * 10
    fmt.Printf("in withArray1: %p, %v\n", &arr, arr)
}

func main() {
    arr := [1]int{1}
    fmt.Printf("before withArray1: %p, %v\n", &arr, arr)
    withArray1(arr)
    fmt.Printf("after withArray1: %p, %v\n", &arr, arr)
}
```

出力:

```
before withArray1: 0xc4200140b8, [1]
in withArray1: 0xc4200140d8, [10]
after withArray1: 0xc4200140b8, [1]
```

特に自分が混乱したのは、`*` で明示的にポインタ型だと宣言するのに対し、slice と map は暗黙的にポインタとして扱われる (正確な理解では無さそう。ただ少なくとも使う側としてはそのように扱えると見て良さそう) というせいだと思います。そこが理解できていないと関数の引数とかに渡していてあれっとなりがちな気がします。

### range で返される 2 番目の値はコピーされた値

[A Tour of Go の Range][4] より

> The range form of the for loop iterates over a slice or map.
> When ranging over a slice, two values are returned for each iteration. The first is the index, and the second is a copy of the element at that index.

```go
package main

import (
    "fmt"
)

func main() {
    xs := []int{1, 2, 3}
    for i, v := range xs {
        fmt.Printf("index: %d, original: %p, value derived from range: %p\n", i, &xs[i], &v)
    }
}
```

出力:

```
index: 0, original: 0xc4200122c0, value derived from range: 0xc4200140b8
index: 1, original: 0xc4200122c8, value derived from range: 0xc4200140b8
index: 2, original: 0xc4200122d0, value derived from range: 0xc4200140b8
```

自分はこの for 文の形式をあまり意識せず使用しており、渡された `v` に変更を加えれば配列の各要素を変更できるよなーと思っていました。そうすると実際渡されているのはただのコピーなので変更は反映されず 1 ハマり獲得です。Go で各要素に変更を加えるような操作をする場合 `for i:= 0; ...` のように添字アクセスするしかなさそうです。

### slice と array は型が違うし配列同士も長さが違えば型が違う

以下のコードでコメントアウトした部分の関数呼出は型が合わずコンパイルできません。

```go
package main

import (
    "fmt"
)

func withArray1(arr [1]int) {
    fmt.Println("withArray1")
}

func withSlice(arr []int) {
    fmt.Println("withSlice")
}

func main() {
    arr1 := [1]int{1}
    arr2 := [2]int{1, 2}
    sli := []int{1}

    withArray1(arr1)
    // cannot use sli (type []int) as type [1]int in argument to withArray1
    // withArray1(sli)
    // cannot use arr2 (type [2]int) as type [1]int in argument to withArray1
    withArray1(arr2)

    withSlice(sli)
    // cannot use arr1 (type [1]int) as type []int in argument to withSlice
    // withSlice(arr1)
}
```

### slice と違い array は append できない

`append` の定義を見るとわかるように第一引数の型は slice なので array を append することはできません。

```
func append(slice []Type, elems ...Type) []Type
```

とはいえ array を使用する場合長さが決め打ちできるという状況のはずなので、そりゃそうだという感じです。

[1]: https://golang.org/doc/effective_go.html
[2]: https://golang.org/doc/effective_go.html#arrays
[3]: https://golang.org/doc/effective_go.html#slices
[4]: https://tour.golang.org/moretypes/16
