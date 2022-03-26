---
layout: post
title: Rust の Future について
tags: "rust"
comments: true
---

最近ある程度まとまった時間を Rust の `Future` 習熟に費やしたのでその備忘録です。

- [基本的な使い方](#basic-usage)
- [Future が実行されるイメージを持つ重要性](#learn-future-mechanism)
- [Future トレイトについて](#trait)
- [async 関数から Future への変換](#conversion)
- [Pinning について](#pinning)
- [Waker について](#waker)
- [Future まわりのユーティリティ](#utility)
- [依存する外部 crate について](#crate)
- [参考](#references)

---

<div id="basic-usage" />

### 基本的な使い方

現在の Rust では非同期処理を `Future` トレイトとして抽象化し、async/await キーワードによるシンタックスシュガーでそれを扱いやすくしています。

```rust
// 事前に tokio crate を依存先に追加している前提

use std::future::Future;

#[tokio::main]
async fn main() {
    let fut = greet1();
    fut.await;
}

async fn greet1() {
    println!("good morning");
}
```

他の async/await を提供するプログラミング言語、例えば JavaScript を知っていると馴染みのあるシンタックスですが、実行に際しては異なる点もあります。

#### Future は明示的に実行する必要がある

例えば JavaScript の `Promise` は生成直後から実行が開始されますが、Rust の `Future` は明示的に実行を指示する必要があります。上記の例では `let fut = greet1()` の時点では `greet1` で定義された処理はまだ実行されておらず、`fut.await` の時点で初めて開始されます。

#### ランタイムを別に用意する必要がある

`Future` はいわば「何を行うか」を定義していますが、それ単体では実行することができずランタイムと呼ばれるものが必要になります。これは他言語の非同期処理実装においてはエグゼキュータやスケジューラと呼ばれたりするものにあたるかと思います。

非同期処理を標準でサポートする言語ではこのランタイムにあたる部分も予め実行環境に含まれていることが多いと思うのですが、Rust の場合は `Future` 自体は標準ライブラリに存在するもののランタイムについては外部 crate から好きな実装を選んで持ってくることになっています。

上記の例では async にした main 関数に `tokio::main` アトリビュートを付与しています。これにより main を tokio の提供するランタイムで実行することを指示しています。main 関数は単純には `tokio::main` を使わずに以下のように書き直すこともでき、こうするとそのことがよりわかりやすくなります。

```rust
fn main() {
    // ランタイムの生成
    let rt = tokio::runtime::Runtime::new().unwrap();
    // greet1 完了までこのスレッドをブロックして待つ
    rt.block_on(greet1());
}
```

<div id="learn-future-mechanism" />

### Future が実行されるイメージを持つ重要性

async/await キーワードのおかげもあり単純な非同期処理ならば容易に `Future` で実装できます。また tokio や async-std といった crate では多くの非同期化された I/O 処理が提供されているため、それらを組み合わせることで多くの非同期処理のニーズも満たせそうです。

そのため `Future` をどのように使うかさえわかればその実装を知らなくても一ユーザとしてはあまり問題なさそうにも思います。とはいえ実際に `Future` を利用していると、コンパイルエラーの内容を掴むのに苦労させられることもあるので、それを解決するためにある程度実装とまではいかなくても実行時の挙動のイメージを知っておくことは有用だと思います。

以降の何節かでは参考にした文献の箇所を紹介しながら `Future` の理解に重要そうな概念や事柄をさらっていきます。

<div id="trait" />

### Future トレイトについて

冒頭に示した async 関数 `greet1` ですがこれは `greet2` のように書き表すこともできます。このように async 関数の実体は `Future` を実装した何らかの型を返す関数です。

```rust
async fn greet1() {
    println!("good morning");
}

// same as greet1
fn greet2() -> impl Future<Output = ()> {
    async {
        println!("good morning");
    }
}
```

`greet2` ではまだ `async` ブロックを使用しており `Future` の実体が見えにくいので、これを更に自分で Future を用意するという形で書き直してみます。

```rust
#[tokio::main]
async fn main() {
    GreetFuture {}.await;
}


// same as greet1
struct GreetFuture {}

impl Future for GreetFuture {
    type Output = ();

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        println!("good morning");
        Poll::Ready(())
    }
}
```

`Future` は `Output` 型と `poll` 関数の定義を求めるトレイトです。`Output` は非同期処理の返り値を表現する型であり、`poll` で行いたい処理を定義します。`poll` は処理が完了した場合に `Poll::Ready(res)` を返し、完了できない場合は `Poll:Pending` を返します。`poll` のシグネチャにでてくる `Pin` や `Context` には後ほど触れます。

[Async/Await - Writing an OS in Rust][3] の「Cooperative Multitasking」項で言及されていることであり、これが個人的にはわかりやすい説明だったのですが、Rust の `Future` の実行モデルは cooperative multitasking として理解できます。ランタイムから poll されることでタスクの処理を進め、`Poll::Pending` を返す (async/await では await で起こり得る) ことでタスク側から処理の途中で自発的に CPU を手放し別のタスク実行をランタイムに促します。

逆にいうと poll 中にブロッキングな処理を行ったり、また await が出現しないような処理の場合、そのタスクが CPU を利用し続け思ったような効率がでない可能性があります。例えば以前勉強として簡単な Blockchain を実装したことがあるのですが、ブロックのマイニング処理を非同期で行うために `Future` として実装したところ、他の非同期処理が全く進まなくなるということを経験しました。このような CPU-bound だったりブロッキングな処理を非同期で行う場合は、ランタイム側で別の仕組みを用意しているはずなのでそういったものを利用することになります。tokio ではそのようなタスクのために [tokio::task::spawn\_blocking][7] が提供されており、実装としてはどうやら新スレッドを用意してそこで実行させているようです。

<div id="conversion" />

### async 関数から Future への変換

async/await を用いたコードから `Future` への変換は、[Async/Await - Writing an OS in Rust][3] の「The Async/Await Pattern」項で記述されている内容が参考になりました。これによれば await 部で状態を遷移する state machine として理解すれば良いようです。

例えば以下のような `example` 関数で考えます。

```rust
// f, g は impl Future<Output=()> を返す関数とする
async fn example() {
    f().await;
    g().await;
}
```

`example` 関数を async/await を使わずに直接 `Future` を実装する形でそれっぽく書き直すと、以下のような状態を持ち、f や g の `Future` が完了することで次の状態に遷移する state machine として理解できます (コードはあくまで雰囲気であり実際にはコンパイルはできない)。

- `Future f` の完了を待つ状態
- `Future g` の完了を待つ状態
- 終了状態

```rust
struct ExampleFuture1 {
    f: FutOne,
    g: FutTwo,
    state: ExampleFuture1State,
}

enum ExampleFuture1State {
    WaitingFut1,
    WaitingFut2,
    Done,
}

impl Future for ExampleFuture1 {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            match self.state {
                ExampleFuture1State::WaitingFut1 => {
                    match self.f.poll(...) {
                        Poll::Ready(_) => {
                            self.state = ExampleFuture1State::WaitingFut2;
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                ExampleFuture1State::WaitingFut2 => {
                    match self.g.poll(...) {
                        Poll::Ready(_) => {
                            self.state = ExampleFuture1State::Done;
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                ExampleFuture1State::Done => return Poll::Ready(()),
            }
        }
    }
}
```

参考:

- [Async/Await - Writing an OS in Rust][3] の The Async/Await Pattern

<div id="pinning" />

### Pinning について

`Future` トレイトが持つ `poll` 関数のパラメータの一つとして登場した `Pin` ですが、これは async ブロック、関数を `Future` に変換する際に生じ得る自己参照を扱うために使われているようです。

ここでいう自己参照とは以下のように自分自身を参照するフィールドを持たせたい構造体によって発生するもので、このときこの構造体が move されると参照が不正になるという問題があります。

```rust
struct PinSample {
    x: i32,
    y: *const i32,
}

pub async fn sample() {
    let mut one = PinSample {
        x: 0,
        y: 0 as *const i32,
    };
    one.y = &one.x as *const i32;

    // one.y は単にポインタなので one が move してもその値は維持されてしまう
}
```

`Pin<&mut T>`, `Pin<&T>`, `Pin<Box<T>>` といった型は T が `Unpin` でない場合にその参照先が (安全には) move されないことを保証します。`Pin` に対する `DerefMut` の定義には以下のように `Unpin` が求められており、これにより `!Unpin` な型 T への参照を含む `Pin` からは安全に T の可変参照 `&mut T` を取り出すことができなくなっています。

```rust
impl<P: DerefMut<Target: Unpin>> DerefMut for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    }
}
```

ただこれは "安全に" 取り出せないというだけで、unsafe な関数 `get_unchecked_mut` 等を使うと操作することは可能です。つまり unsafe なしでは move されないことがコンパイラによって保証される一方、unsafe に操作する箇所ではそれをユーザが保証する必要があります。

`Unpin` は [auto trait][10] であり、いまのところ実用上出現する `Unpin` でない型は async/await により作られた `Future` ぐらいなようです。

先程の `ExampleFuture1` を pinning を考慮してコンパイルできる形に書き直すとこうなります。実装には [futures crate の Map][8] を参考にしているので大きく間違ったことはしていないと思いますが細かいところは怪しいかもです。なおスタック上への pinning には pin-utils の代わりに [pin-project][9] を利用しています。

```rust
use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// FutOne, FutTwo は Future を実装した型
#[pin_project]
pub struct ExampleFuture1 {
    #[pin]
    f: FutOne,
    #[pin]
    g: FutTwo,
    state: ExampleFuture1State,
}

enum ExampleFuture1State {
    Start,
    WaitingFut1,
    WaitingFut2,
    Done,
}

impl ExampleFuture1 {
    pub fn new() -> ExampleFuture1 {
        ExampleFuture1 {
            f: FutOne {},
            g: FutTwo {},
            state: ExampleFuture1State::Start,
        }
    }
}

impl Future for ExampleFuture1 {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            let x = self.as_mut().project();
            match x.state {
                ExampleFuture1State::Start => {
                    *x.state = ExampleFuture1State::WaitingFut1;
                }
                ExampleFuture1State::WaitingFut1 => match x.f.poll(cx) {
                    Poll::Ready(_) => {
                        *x.state = ExampleFuture1State::WaitingFut2;
                    }
                    Poll::Pending => {
                        return Poll::Pending;
                    }
                },
                ExampleFuture1State::WaitingFut2 => match x.g.poll(cx) {
                    Poll::Ready(_) => {
                        *x.state = ExampleFuture1State::Done;
                    }
                    Poll::Pending => {
                        return Poll::Pending;
                    }
                },
                ExampleFuture1State::Done => return Poll::Ready(()),
            }
        }
    }
}
```

参考:

- [Asynchronous Programming in Rust][2] の Pinning
- [Async/Await - Writing an OS in Rust][3] の Pinning
- [RustのPinチョットワカル - OPTiM TECH BLOG][4]

<div id="waker" />

### Waker について

`Future` の `poll` 関数には `cx: &mut Context<'_>` というパラメータを介して [Waker][11] が渡されます。一ユーザとして用意された非同期処理を async/await で組み合わせていくだけだとあまり意識されない要素ですが、`Waker` は各 `Future` を次にいつ poll してほしいかをランタイムに知らせるための仕組みです。ランタイムが一度 `Future` を poll したあと、対象の処理が進んだタイミングで `Waker` を介して再度の poll を依頼することで、より効率的に処理を進めることができます。

例えば tokio の提供する非同期ネットワーク I/O では (Linux の場合) epoll により適切なタイミングで wake させているようです。また同様に epoll を利用した `Future` 実装例は参考先の [並行プログラミング入門][6] でも見つかります。

余談ですが tokio の非同期 File I/O は `spawn_blocking` つまり別スレッドでブロッキングな処理を走らせるという実装になっています。これは epoll が File I/O を対象にすることができず、また io\_uring のような仕組みは対象カーネルのバージョンが限定されすぎるといったことがあり、どうやら Linux においては File I/O を非同期に処理するために何を利用するかというのは難しい問題みたいです。

参考:

- [Asynchronous Programming in Rust][2] の Executro with Waker Support
- [並行プログラミング入門][6] の 5 章
- [Linuxにおける非同期IOの実装について][12]

<div id="utility" />

### Future まわりのユーティリティ

ここまでで `Future` の動作を理解するために必要と思われる概念を一通り見てきたので、次は実際に `Future` を使った処理を書く上で「これどう書けばいいんだっけ」となりそうなものについて触れていきます。

#### join

join は複数の `Future` の結果を待ち受けて処理を行うためのユーティリティです。

例えば複数の外部サービスに情報を問い合わせて結果をまとめる必要があり、かつ個々のリクエストが独立しているならばそれら全部を先に実行開始して結果だけを待ち受けることで効率的に処理を行うことができます。このような処理は join 関数、マクロとして各 crate が提供しています。

以下は futures の提供する join を使用した例です。

```rust
use std::time::Duration;

async fn request_to_service1() -> u64 {
    tokio::time::sleep(Duration::from_secs(3)).await;
    3
}

async fn request_to_service2() -> u64 {
    tokio::time::sleep(Duration::from_secs(5)).await;
    5
}

// これだと service1, service2 へのリクエストが順に行われるので 3 + 5 秒かかる
pub async fn sample1() {
    let res1 = request_to_service1().await;
    let res2 = request_to_service2().await;
    println!("{}, {}", res1, res2);
}

// これだと 5 秒で済む
// 先に task を spawn しておく必要あり
// (実際は直接 Future 渡しても問題無さそうだが join の実装次第になってしまうはず)
pub async fn sample2() {
    let h1 = tokio::task::spawn(request_to_service1());
    let h2 = tokio::task::spawn(request_to_service2());
    let res = futures::future::join(h1, h2).await;
    println!("{:?}", res);
}
```

join の実装ですが 2 つの `Future` を待ち受けるような処理は概念的には以下のように書けます。

```rust
use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

#[pin_project]
struct MyJoin<T: Future, U: Future> {
    #[pin]
    fut1: T,
    #[pin]
    fut2: U,
}


// 2 つの Future を待ち受ける Future
// あくまで擬似的な実装であることに注意。
// Future の poll は Ready を返したあと再度呼ばれた時の挙動が未定義なので実際は実行時に panic する。
impl<T: Future, U: Future> MyJoin<T, U> {
    pub fn new(fut1: T, fut2: U) -> MyJoin<T, U> {
        MyJoin { fut1, fut2 }
    }
}

impl<T: Future, U: Future> Future for MyJoin<T, U> {
    type Output = (T::Output, U::Output);

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let futures = self.project();
        let res1: Poll<T::Output> = futures.fut1.poll(cx);
        let res2: Poll<U::Output> = futures.fut2.poll(cx);
        match (res1, res2) {
            (Poll::Ready(res1), Poll::Ready(res2)) => Poll::Ready((res1, res2)),
            _ => Poll::Pending,
        }
    }
}

pub async fn sample() {
    let h1 = tokio::task::spawn(async {
        tokio::time::sleep(Duration::from_secs(3)).await;
        println!("fut1");
    });

    let h2 = tokio::task::spawn(async {
        tokio::time::sleep(Duration::from_secs(5)).await;
        println!("fut2");
    });

    MyJoin::new(h1, h2).await;
}
```

ただしコメント中にも記載した通りこれはあくまで擬似的な実装なので、実際のコードは例えば [futures の join\_all][13] を見るのが良いです。こちらでは `MaybeDone` という結果を取り出すまでは poll を再度呼び出すことができる `Future` を用意したり、任意数の `Future` を待ち受けることができるようになっています。

#### select

いくつかの同じ型の結果を返す `Future` から一つだけ結果が取れればいいという場面では各種 crate が提供する `select` 関数、マクロが使用できます。以下は futures の `select` 使用例です。

```rust
use futures::FutureExt;
use std::time::Duration;

async fn f(i: u64) -> u64 {
    tokio::time::sleep(Duration::from_secs(i)).await;
    println!("finish {}", i);
    i
}

pub async fn sample() {
    // f(3) がはじめに完了するはず
    let h1 = tokio::spawn(f(5));
    let h2 = tokio::spawn(f(4));
    let h3 = tokio::spawn(f(3));
    let (res, remaining) = futures::future::select_ok(vec![h1, h2, h3]).await.unwrap();
    println!("{:?}", res);

    // f(5), f(4) については h1, h2 を drop すればそれ以降実行は進まない
    // あるいは明示的に abort する
    // (とはいえタスク目線だと abort されていることに気付くのは await しているタイミングっぽいので
    // この例だと結局最後まで実行される)
    for h in remaining {
        h.abort();
    }
}
```

`select` の挙動は比較的わかりやすく 2 つの `Future` を select するような `Future` を自前で実装すると以下のようになります。ただしこれも簡便に実装するために futures のそれより機能を減らしています (例えば残りの未完了の `Future` を返すには `select` に渡す `Future` を `Unpin` にする必要が出てくる)。

```rust
use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

#[pin_project]
struct MySelect<T: Future> {
    #[pin]
    fut1: T,
    #[pin]
    fut2: T,
}

impl<T: Future> MySelect<T> {
    pub fn new(fut1: T, fut2: T) -> MySelect<T> {
        MySelect { fut1, fut2 }
    }
}

impl<T: Future> Future for MySelect<T> {
    type Output = T::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let futures = self.project();
        if let Poll::Ready(res) = futures.fut1.poll(cx) {
            return Poll::Ready(res);
        }
        if let Poll::Ready(res) = futures.fut2.poll(cx) {
            return Poll::Ready(res);
        }
        Poll::Pending
    }
}

pub async fn sample() {
    let h1 = tokio::task::spawn(async {
        tokio::time::sleep(Duration::from_secs(3)).await;
        println!("fut1");
    });

    let h2 = tokio::task::spawn(async {
        tokio::time::sleep(Duration::from_secs(5)).await;
        println!("fut2");
    });

    // should print "fut1"
    MySelect::new(h1, h2).await;
}
```

#### recursion

例えば (非同期に書く必要はありませんが) 階乗の計算を async/await で書いてみます。

```rust
async fn factorial(x: u32) -> u32 {
    return if x == 0 {
        1
    } else {
        x * factorial(x - 1).await
    };
}
```

すると以下のようなコンパイルエラーが発生します。

```
error[E0733]: recursion in an `async fn` requires boxing
--> src/sample_rec.rs:1:31
|
1 | async fn factorial(x: u32) -> u32 {
    |                               ^^^ recursive `async fn`
    |
    = note: a recursive `async fn` must be rewritten to return a boxed `dyn Future`
    = note: consider using the `async_recursion` crate: https://crates.io/crates/async_recursion
```

どうやらこれは async/await から `Future` への変換時に自分自身を含む構造体の定義が生成されてしまい、そのサイズが決定できないために失敗しているようです。いわば以下のような構造体を生成できず間接的に `Box` 等で参照する必要がある、という場面と同じ話なのだと思います。

```rust
// これだとサイズが決まらないので、
struct Foo {
    foo: Box<Foo>,
}


// 例えばこうする必要がある
struct Foo {
    foo: Box<Foo>,
}
```

ここでは futures が提供する `BoxFuture` に変換することで解決させます。

```rust
use futures::future::BoxFuture;
use futures::FutureExt;

// in futures-core
// pub type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

fn factorial(x: u32) -> BoxFuture<'static, u32> {
    async move {
        return if x == 0 {
            1
        } else {
            x * factorial(x - 1).await
        };
    }
    .boxed()
}
```

はじめ単純に `Box` で作成した `Future` を包めばいいのではと思ったのですが `Box<F, A>` から `Future` を導出するには F が `Future + Unpin` である必要があります。しかし async/await から変換された `Future` は `!Unpin` です。

```rust
// in alloc::boxed

#[stable(feature = "futures_api", since = "1.36.0")]
impl<F: ?Sized + Future + Unpin, A: Allocator> Future for Box<F, A>
where
    A: 'static,
{
    type Output = F::Output;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        F::poll(Pin::new(&mut *self), cx)
    }
}
```

そのため更に `Pin` で包むことで `Box` に定義された `DerefMut` と合わせて下の定義により `Future` を導出しています。

```rust
// in std::future::Future

#[stable(feature = "futures_api", since = "1.36.0")]
impl<P> Future for Pin<P>
where
    P: ops::DerefMut<Target: Future>,
{
    type Output = <<P as ops::Deref>::Target as Future>::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        <P::Target as Future>::poll(self.as_deref_mut(), cx)
    }
}
```

同様なことは `async-recursion` が提供するアトリビュートによっても行えます。

参考:

- [Asynchronous Programming in Rust][2] の 7.3 Recursion

#### schedule

遅延させて、または定期的に実行したい `Future` がある場合は例えば tokio の `tokio::time::sleep` や `tokio::time::interval` を利用できます。

```rust
use std::time::Duration;

// 一秒に一回処理を行う
// 処理の間は必ず 1 秒空けられる
async fn f() {
    loop {
        tokio::time::sleep(Duration::from_secs(1)).await;

        // do what we want here
        println!("do f");
    }
}

// 一秒に一回処理を行う
// 処理に 1 秒以上の時間がかかった場合はすぐに処理が行われる
async fn g() {
    let mut interval = tokio::time::interval(Duration::from_secs(1));
    loop {
        interval.tick().await;

        // do what we want here
        println!("do g");
    }
}
```

コメントで記載したように 2 つの動作は端的にはどちらも「一秒に一回処理を行う」なのですがその間隔は微妙に異なる可能性があります。わかりやすい例としては行いたい処理として `std::thread::sleep(Duration::from_secs(2))` を追加すると、Future f の場合は 3 秒毎に "do f" が表示されますが、Future g の場合はそれが 2 秒毎になります。

<div id="crate" />

### 依存する外部 crate について

ここまで見てきたように現在の Rust で `Future` を使用した非同期処理を書く際には外部 crate を利用するのが一般的です。自分がはじめて Rust の `Future` に触れたとき、どの crate を利用するべきか、また複数 crate に依存するのはありなのか悩んだのですが、いまの自分の結論だと futures と tokio、あるいは futures と async-std の組み合わせを候補にするのがいいのかなと思っています。

`Future` に関して外部 crate を使いたいモチベーションとしては、

- ランタイムが欲しい
- 非同期 I/O 処理のサポートが欲しい
- `Future` のユーティリティ (e.g. futures の提供する Stream) が欲しい

といったあたりが挙げられます。基本的には 1, 2 番目の項目は crate 依存になる部分であり、3 番目については異なる crate から持ってきても利用できる部分になるはず (見極めが難しいですが...) です。

ユーティリティ目的として将来的に標準ライブラリにも入るかもしれない futures を入れつつ、ランタイムや非同期 I/O 目的に tokio か async-std といったメジャーなものを使うのが良いのかなと考えています。

この疑問については [The Async Ecosystem - rust-lang.github.io/async-book][1] でまさに語られている内容でもあるので、気になる方は一読すると良いかもしれません。

<div id="references" />

### 参考

- [Asynchronous Programming in Rust][2]
  - Future を中心とした Rust の非同期処理関する公式ドキュメント
  - まずはこれを読むのが良さそうです
- [Async/Await - Writing an OS in Rust][3]
  - 前半で Rust の Future についての説明が、後半ではそれを自作 OS に組み込む説明
  - 多少前提知識が必要な気もしますが、Future の実装理解には一番役立ちました
- [実践Rustプログラミング入門][5]
  - 非同期処理も含め記述された Rust 入門書
  - Future については全体像を理解するのに役立ちました
- [並行プログラミング入門][6]
  - 並行プログラミングに必要な知識をまとめた本
  - 5 章に epoll を Future で抽象化した実装を行う部分があり、そこが理解に役立ちました
- [RustのPinチョットワカル - OPTiM TECH BLOG][4]
  - Rust の Pin についての解説
  - そもそも Pin ってなんなのというところから日本語で解説されています

[1]: https://rust-lang.github.io/async-book/08_ecosystem/00_chapter.html
[2]: https://rust-lang.github.io/async-book/
[3]: https://os.phil-opp.com/async-await/
[4]: https://tech-blog.optim.co.jp/entry/2020/03/05/160000
[5]: https://www.shuwasystem.co.jp/book/9784798061702.html
[6]: https://www.oreilly.co.jp/books/9784873119595/
[7]: https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html
[8]: https://docs.rs/futures-util/0.3.4/src/futures_util/future/future/map.rs.html
[9]: https://docs.rs/pin-project/1.0.10/pin_project/
[10]: https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits
[11]: https://doc.rust-lang.org/std/task/struct.Waker.html
[12]: https://qiita.com/tmsn/items/0b9e5f84f9fbc56c1c82
[13]: https://github.com/rust-lang/futures-rs/blob/master/futures-util/src/future/join_all.rs
