---
layout: post
title: Rust で順列生成する Iterator の実装
tags: "rust"
comments: true
---

Rust 習熟のために競プロや [rust99][2] をちょこちょこやっているのですが、その中で順列を生成する Iterator が欲しいなと思う場面があったので [permutation\_iterator.rs][1] で実装してみました。

順列を生成する Iterator というのを使用例で示すと、

```rust
for p in vec![1, 2, 3].permutation() {
    println!("{:?}", p);
}
```

で

```
[1, 2, 3]
[1, 3, 2]
[2, 1, 3]
[2, 3, 1]
[3, 1, 2]
[3, 2, 1]
```

を生成するようなものです。

実装中のコメントにも書いていますが順列生成のアルゴリズム自体は C++ の [next\_permutation][3] を参考にしています。
このアルゴリズムはパッと見だと難しく感じますが自分で具体的にシミュレーションすると雰囲気は理解できると思います。

Rust 的な話でいうと `permutation` メソッドを提供するために `Permutation` トレイトを用意し、それを `IntoIterator` から導出できるようにしたのですが、それが思ったより使い心地が良かったです。

```rust
pub trait Permutation<T: Ord + Clone> {
    fn permutation(self) -> PermutationIterator<T>;
}

// IntoIterator を実装するものに対して Permutation を実装する
impl <T: Ord + Clone, I: IntoIterator<Item=T>> Permutation<T> for I {
    fn permutation(self) -> PermutationIterator<T> {
        PermutationIterator::new(self.into_iter().collect())
    }
}
```

これで `vec![1, 2, 3]` のようなものに permutation が使えるのはわかっていたのですが、各要素に Clone を実装していない場合でも同じように `Vec<&T>` として結果を返してくれます。 

```rust
// i32 を単にラップしたもの
// Eq や Ord の定義は i32 に移譲している (ここでは省略)
// Clone は未実装
struct MyInt(i32);

let xs = vec![MyInt(1), MyInt(2), MyInt(3)];

// ここで欲しいのは &MyFoo なので
// impl<T, A: Allocator> IntoIterator for Vec<T, A> ではなく
// impl<'a, T, A: Allocator> IntoIterator for &'a Vec<T, A> を使っている?
for p in xs.permutation() {
    println!("{}, {:?}", type_of(&p), p);
}
```

```
alloc::vec::Vec<&main::MyInt>, [MyInt(1), MyInt(2), MyInt(3)]
alloc::vec::Vec<&main::MyInt>, [MyInt(1), MyInt(3), MyInt(2)]
alloc::vec::Vec<&main::MyInt>, [MyInt(2), MyInt(1), MyInt(3)]
alloc::vec::Vec<&main::MyInt>, [MyInt(2), MyInt(3), MyInt(1)]
alloc::vec::Vec<&main::MyInt>, [MyInt(3), MyInt(1), MyInt(2)]
alloc::vec::Vec<&main::MyInt>, [MyInt(3), MyInt(2), MyInt(1)]
```

[1]: https://gist.github.com/tiqwab/94048e636a1a01760f1fe52977083be5
[2]: https://github.com/mocobeta/rust99
[3]: https://cpprefjp.github.io/reference/algorithm/next_permutation.html
