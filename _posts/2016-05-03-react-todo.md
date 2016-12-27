---
layout: post
title:  "Thinking in Reactに従いながらTODOアプリ作成"
tags: "javascript, react"
comments: true
---

JavaScriptのライブラリ、Reactの試し書き。Reactを使用したWebアプリの実装手順を整理したいため、React公式のdocumentation, [Thinking in React][1]を参考にしながら、TODOアプリを作成してみる。

### Start with a mock

TODOアプリのmockを作成しておく。TODOリストと新規TODO追加用のフォーム、また状態毎のTODOを表示するためのセレクタを配置してみた。デザイナーでは無いのでそんなに見た目イケイケなものは作れない。

<img
  src="/images/react-todo/todo-app-mock.png"
  title="Mock of todo app"
  alt="Mock of todo app"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

### Step 1: Break the UI into a component hierarchy

設計したUIをプログラムに落とし込む前準備。

- UIをcomponentが階層上に構成されたものとみなし、それぞれのcomponentに名前をつける。上でmockを作っているならそのレイヤー名なんかを使えばよろし。
- コンポーネントの分け方は通常のプログラミング同様、単一責任の原則([signle responsibility principle][2])に従うべき。
- ModelとUIは同様の構造を持つようになるだろうし、そうであるべき。

今回のtodo appでは以下のようにcomponentを定義し作成していく。

<img
  src="/images/react-todo/todo-app-component-hierarchy.png"
  title="Component hierarchy of todo app"
  alt="Component hierarchy of todo app"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

### Step 2: Build a static version in React

Reactでstatic versionのUIを作る。

- static versionということは表示に使用するデータをAPI等外部とのやりとりではなく、プログラム上やファイルにベタ書きしたものを使ってまずは、ということ。
- staticという前提上、このステップでReactのcomponentを作る際には、**propを使い**、**stateは使わない**。

### Step 3: Identify the minimal (but complete) representation of UI state

Step 2で作ったstatic UIをインタラクティブなものにするために、何を`state`にするかを判断する。

- 必要最小限なものを選ぶ。以下のようなデータは`state`とはみなさない。
  - 親componentから`prop`で渡されるもの
  - 時間に依らず不変なもの
  - 他の`prop`や`state`から導けるもの (e.g. TODOの個数)

以上を踏まえて、今回のTODOアプリでは以下のものを`state`とする。

- 新規TODO入力フォームの内容
- TODOリスト
- 表示するTODOの種類を決めるためのセレクタ

### Step 4: Identify where your state should live

Step 3で定めた`state`をどのcomponentに持たせるかを決める。決め方は

1. `state`を描画するcomponentがどれかを判断する。
2. 1.で見つけたcomponentに共通の親componentを探す。
3. 2. のcomponentあるいは更にその親となるようなcomponentに`state`を持たせる。

という感じ。もし3.に当てはまるようなcomponentが無い場合は、新規に親componentを作ってしまえば良い。  

今回は`TodoBox`に`state`を所有させるのが適当そう。

### Step 5: Add inverse data flow

Step 4の状態では`state`を表示することはできるが、画面から`state`を変更することができない(変更が反映されない)。なので`onChange`等のイベントハンドリングを実装して双方向のデータバインディングが行われるようにする。

### 作成時詰まった点

- componentの`state`を変更する時には`setState`メソッドの使用が基本みたい
  - `state`はimmutableなものとして扱うべきとのこと。
- `Todo` componentで発生したイベント処理の方法。対象todoを取得したい場合にどうするべきか。
  - 今回は以下のように`TodoList` componentが`Todo`を生成するときに、その`Todo`をbindしたイベントハンドラを用意することで対応した。想定通りに機能しているが、`render`メソッド中で毎回bindしているのはムダがある？

```javascript
render() {
  ...
  const onToggle = this.onUserToggle.bind(this, x);
  const deleteTodo = this.deleteTodo.bind(this, x);
  return (
    <Todo
      key={x.id}
      todoId={x.id}
      title={x.title}
      completed={x.completed}
      onToggle={onToggle}
      deleteTodo={deleteTodo}
    />
  );
}
```

### サンプル用リポジトリ

- [react-todo][3]

[1]: https://facebook.github.io/react/docs/thinking-in-react.html
[2]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[3]: https://github.com/tiqwab/react-todo
