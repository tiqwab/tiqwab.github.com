---
layout: post
title:  "JavaScript学び　-開発ツール関連-"
tags: "javascript, node, npm, eslint, babel, webpack"
comments: true
---

JavaScript界隈の知識があまり無かったのでまとめてみるその1。ここではよく聞く開発ツールのインストールや使用法について確認する。

1. [パッケージマネージャnpm](#anchor1)
2. [ESLintとAirbnb JavaScript Style Guideによる構文チェック](#anchor2)
3. [BabelによるJavaScriptトランスパイル](#anchor3)
4. [webpackによるビルド](#anchor4)
5. [webpack + Babel](#anchor5)

<a id="anchor1"></a>

### 1. パッケージマネージャnpm

JavaScriptのパッケージマネージャnpmとそれを動かすためにnode.jsをHomebrewでインストール。と同時にnode.jsとかnpmとか今まで漠然とした理解しか持っていなかったので、[こちら][2]の記事を見て少し頭を整理。

#### インストール

```bash
$ brew install node
```

npmでは`package.json`が依存性の管理を行っている。JavaだとMavenの`pom.xml`的な。  
npmに関してはとりあえず以下のコマンドだけ使えれば何とかなりそう。

- `npm init --yes`
- `npm install <package_name> --save`
- `npm install <package_name> --save-dev`
- `npm update`
- `npm uninstall <package_name> --save`
- `npm uninstall <package_name> --save-dev`
- `npm install -g <package_name>`

[こちら][3]によれば、パッケージのバージョンは`x`, `1.x`, `1.1.x`の様な形式で指定することも可能。  

また必須ではないが、利便性のためここでは`npm-run`もglobally installしておく。

```bash
$ npm install -g npm-run
```

参考

- [npm とか bower とか一体何なんだよ！Javascript 界隈の文脈を理解しよう][2]
- [npm Documentation - semantic versioning and npm][3]

<a id="anchor2"></a>

### 2. ESLintとAirbnb JavaScript Style Guideによる構文チェック

JavaScriptのECMAScript 6(ES6)仕様に基づく文法、構文チェックをAtomエディタ上で機能させるための準備。手順は以下の通り。

1. `package.json`の作成
2. ESLintを`npm install`
3. Airbnbのチェックスタイル使用のためのパッケージを`npm install`
4. Atomエディタ上でチェックを行うためのプラグインを`apm install`

#### インストール

```bash
$ npm init --yes

$ npm install -g eslint
$ eslint --init

$ npm install --save-dev eslint-config-airbnb eslint-plugin-import eslint-plugin-react eslint-plugin-jsx-a11y eslint

$ apm install linter
$ apm install linter-eslint
```

2, 3に関してはよく使用するなら全部`npm install -g`で良いのかもしれないが、今回はとりあえず2だけそうしている。なお、3.のインストール時に、`UNMET PEER DEPENDENCY`という警告が出されてしまうが、[Airbnb JavaScript Style Guide][6]の通りにチェックが走っているようなのでとりあえず今は放置。

参考

- [ReactのチュートリアルをES6で書いてwebpackとESLintも使ってみる][4]
- [eslint-config-airbnb][5]

<a id="anchor3"></a>

### 3. BabelによるJavaScriptトランスパイル

上でES6に基づく文法チェックが機能するように設定したが、現状ではES6の全ての仕様が全てのブラウザで実装されているわけではないので、BabelによりES5仕様へとトランスパイルする。本来は別途タスクツール等と組み合わせて使うと便利なようだが、まずはBabel自体を知りたかったのでコマンドライン上で実行させてみる。

#### インストール

```bash
$ npm install --save-dev babel-cli
```

これでBabelがローカル(プロジェクト)にインストールされ、`package.json`も更新される。コマンドラインから実行するため、`package.json`に以下を追記する。この場合、srcディレクトリ内のファイルが処理され、jsディレクトリに保存される。

```
...
"scripts": {
  "build": "babel src -d js"
},
...
```

以下のコマンドで実行確認。

```bash
$ npm run build
```

なお、上で`npm-run`をインストールしている場合には`npm-run babel src -d js`で同様に実行できる。  
この時点ではBabelに何も処理を行うよう設定していないため、ただファイルをコピーするだけとなる。今回はES6からES5へのトランスパイルとReactへの対応(JSXのコンパイル？)を行わせる想定で、以下の手順を行う。

#### `.babelrc`ファイルの作成

Babelの設定ファイル。

```
{
  "presets": ["es2015", "react"],
  "plugins": ["transform-runtime"]
}
```

#### Preset, Pluginのインストール

```bash
$ npm install --save-dev babel-preset-es2015
$ npm install --save-dev babel-preset-react
$ npm install --save babel-polyfill
$ npm install --save babel-runtime
$ npm install --save-dev babel-plugin-transform-runtime
```

これでもう一度`$ npm run build`を実行させ、トランスパイルが機能していることを確認。

```javascript
import "babel-polyfill";

// New function syntax in es6
const factorial = (x) => {
  if (x == 0) {
    return 1;
  }
  return x * factorial(x-1);
};
console.log(factorial(5));

// A function 'reduce' does not exist in es5
const addAll = (...args) => {
  return args.reduce((a, b) => a + b);
};
console.log(addAll(1, 2, 3, 4, 5));

// New class syntax in es6
class Person {
  constructor(name, sex) {
    this.name = name;
    this.sex = sex;
  }
  hello() {
    return `Hello, my name is ${this.name}`;
  }
}
console.log(new Person('Taro', 'M').hello());
```

↓

```javascript
'use strict';

var _classCallCheck2 = require('babel-runtime/helpers/classCallCheck');

var _classCallCheck3 = _interopRequireDefault(_classCallCheck2);

var _createClass2 = require('babel-runtime/helpers/createClass');

var _createClass3 = _interopRequireDefault(_createClass2);

require('babel-polyfill');

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { default: obj }; }

// New function syntax in es6
var factorial = function factorial(x) {
  if (x == 0) {
    return 1;
  }
  return x * factorial(x - 1);
};
console.log(factorial(5));

// A function 'reduce' does not exists in es5
var addAll = function addAll() {
  for (var _len = arguments.length, args = Array(_len), _key = 0; _key < _len; _key++) {
    args[_key] = arguments[_key];
  }

  return args.reduce(function (a, b) {
    return a + b;
  });
};
console.log(addAll(1, 2, 3, 4, 5));

// New class syntax in es6

var Person = function () {
  function Person(name, sex) {
    (0, _classCallCheck3.default)(this, Person);

    this.name = name;
    this.sex = sex;
  }

  (0, _createClass3.default)(Person, [{
    key: 'hello',
    value: function hello() {
      return 'Hello, my name is ' + this.name;
    }
  }]);
  return Person;
}();

console.log(new Person('Taro', 'M').hello());
```

#### babel-polyfillとbabel-runtime

上では`babel-polyfill`と`babel-runtime`の両方をインストールしていたが、実際は片方のみをインストールするべきのよう？ただし、どちらが良いかはケースバイケース。

参考:

- [babel babel-handbook][7]
- [babel-polyfillとbabel-runtimeの使い分けに迷ったので調べた][12]

<a id="anchor4"></a>

### 4. webpackによるビルド

JavaScriptやCSS等リソースの依存関係を解決しビルドするツール。ビルド時には依存関係の解決だけではなく、設定により他のことも行わせることができる(e.g. Babelによるトランスパイル)。Babelと組み合わせた使用は後述とし、ここではwebpack単体で使ってみる。

#### webpackのインストール

```bash
$ npm install webpack --save-dev
```

#### Loaderのインストール

webpackをインストールしているだけでもJavaScriptファイルを処理することは可能だが、それ以外のjsonやcssといったファイルもLoaderをインストールすることで対応可能となる。

```bash
$ npm install css-loader style-loader --save-dev
```

#### webpack.config.jsの作成

コマンドラインからオプションを指定しまくって直に実行もできるようだが設定が多いと辛いため、通常は`webpack.config.js`に設定を書いてwebpackに参照させるよう。

```javascript
module.exports = {
    entry: {
      bundle: __dirname + "/src/entry1.js",
      other: __dirname + "/src/entry2.js"
    },
    output: {
        path: __dirname + '/js',
        filename: "[name].js"
    },
    module: {
        loaders: [
            { test: /\.css$/, loader: "style!css" }
        ]
    },
    resolve: {
      extensions: ["", ".webpack.js", ".web.js", ".js", "css"]
    }
};
```

- `entry`に指定したファイルをbundle化し、`output`に指定したファイルとして出力。
- `entry`は複数指定できるので、例えば異なるページで異なるbundleを作成することができる。その際`output.filename`には`[name].js`のように指定することでファイル名に`entry`のkeyが埋め込まれる。
- 他にもコードを共通化してpluginとして参照できるようにしたり、非同期読み込みの対応もあるようだけど今は省略。詳細は[petehunt webpack-howto][9]。

#### webpackの実行

`webpack.config.js`をもとにbundleを作成させる。

```bash
$ npm-run webpack
```

`./js/bundle.js`, `./other.js`が作成されていることを確認。

#### 開発サーバの使用

webpack-dev-serverを使用することで開発を行いやすくする。アドレスは`http://localhost:8080/webpack-dev-server/`のようになる。

```bash
$ npm install webpack-dev-server --save-dev
$ npm-run webpack-dev-server
```

参考:

- [webpack getting-started][8]
- [petehunt webpack-howto][9]

<a id="anchor5"></a>

### 5. webpack + Babel

webpackにBabel-loaderを設定し、bundle化の際にトランスパイルさせる。

#### 各種インストール

新規プロジェクト用ディレクトリを作成し、各種ライブラリをインストールする。

```bash
$ npm init --yes
$ npm install --save-dev webpack babel-loader babel-preset-react babel-preset-es2015 babel-plugin-transform-runtime
$ npm install --save babel-polyfill babel-runtime
```

#### webpack.config.js作成

LoaderとしてBabelを追加している。

```javascript
module.exports = {
    entry: {
      app: __dirname + "/src/entry.js"
    },
    output: {
        path: __dirname + '/js',
        filename: "[name].js"
    },
    module: {
        loaders: [
          {
            test: /\.js[x]?$/,
            exclude: /node_modules/,
            loader: "babel",
            query:{
              presets: ['react', 'es2015'],
              plugins: ['transform-runtime']
            }
          }
        ]
    },
    resolve: {
      extensions: ["", ".webpack.js", ".web.js", ".js", "jsx"]
    }
};
```

#### webpackの実行

```bash
$ npm-run webpack
```

上記「3. BabelによるJavaScriptトランスパイル」で使用したJavaScriptを流用したが、ちゃんと処理されている。Babel単体で使用していた時は、babel-polyfillをimportさせていたが、ここでは明示的にimportを指定する必要は無いよう。  
またbabel-loaderのtransform-runtimeプラグインは必須ではなく、Babelが使用するヘルパーメソッドを共通化させるかどうかを決めているだけである。

参考:

- [webpack+babel環境でフロントエンドもES6開発][10]
- [babel babel-loader][11]

### 印象

ぱっと見だとJavaScriptのフロントエンド開発は色々ツールがあって大変だ！と思ってしまうけど、それぞれのツールの行うこと自体はシンプルなので理解しやすいし、逆に組み合わせて使いやすいのかなと感じた。  
フロントエンドをJavaScriptでガシガシ開発する時代がそんな長続きするとは思わない(含む半分願望)けど、最低限はこなせるようにしておきたいとこ。

### サンプル用リポジトリ

- [eslint-example](https://github.com/tiqwab/js-example/tree/master/eslint-example)
- [babel-example](https://github.com/tiqwab/js-example/tree/master/babel-example)
- [webpack-example](https://github.com/tiqwab/js-example/tree/master/webpack-example)
- [babel-webpack-example](https://github.com/tiqwab/js-example/tree/master/babel-webpack-example)

[1]: http://d.hatena.ne.jp/tomoya/20160403/1459665374
[2]: http://qiita.com/megane42/items/2ab6ffd866c3f2fda066
[3]: https://docs.npmjs.com/getting-started/semantic-versioning
[4]: http://qiita.com/morizotter/items/9e2a7def6773a2a8e174
[5]: https://github.com/airbnb/javascript/tree/master/packages/eslint-config-airbnb
[6]: https://github.com/airbnb/javascript
[7]: https://github.com/thejameskyle/babel-handbook/blob/master/translations/en/user-handbook.md
[8]: http://webpack.github.io/docs/tutorials/getting-started/
[9]: https://github.com/petehunt/webpack-howto
[10]: http://qiita.com/HayneRyo/items/74892d3a37ee96a5df60
[11]: https://github.com/babel/babel-loader
[12]: http://qiita.com/inuscript/items/d2a9d5d4daedaacff924
