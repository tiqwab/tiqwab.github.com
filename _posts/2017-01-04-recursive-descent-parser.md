---
layout: post
title:  "数式の再帰的下向き構文解析"
tags: "parser, recursive descent, python"
comments: true
---

構文解析を行う際、世の中にはyaccのように生成規則から構文解析器を生成してもらえるツールが数多くあります。しかしそういったツールを使うまでは無いけど、自分でさっと作りたいなと思うときがあり、最近でいうとHaskellのQuasiQuotes周りでそういった場面がありました。

自分で書くのであれば、再帰的下向き構文解析がやりやすいだろうというのが通説のようなので、その背景と実装方法についてまとめてみます。

### 1. 再帰的下向き構文解析

再帰的下向き構文解析に関する概要を箇条書きでまとめました。

- [文脈自由文法][8]に対する構文解析法の一つ
- 下向き -> 構文木を考えたときに、root側からleafへと解析が進む
  - 出現するtokenと構文規則を元に何を解析するのか予想しながら下へ進む
    - 予想する、という表現を最初見たときに少ししっくりこなかったが、それは恐らくLL(k)で考えているため
    - LL(k)文法であれば、k個のtokenを読むことで次に適用する構文規則が判断できるが、そうでない場合予想して進み間違いだとわかれば戻る(backtracking)必要がでてくる
    - backtrackingを必要とする場合、指数関数的に時間がかかるかも
- 再帰 -> 関数の再帰で実装する
- (上向きに比べれば) 直感的に手書きで書きやすい
  - 各生成規則をそのまま一つの関数に変換してあげる方針でいける
  - 特に、通常の数式や複雑な構文を持たない言語であればLL(1)と呼ばれる文法になり、手書きで解析しやすい

#### LL(k)文法

- 文脈自由文法でかつk文字先読みすれば次に解析するものが何か判断できる。
  - backtrackをしなくて済む。
- もちろんLL(k)で解析できない文脈自由文法も多々存在する

ここではLL(1)文法で表現される数式を言語として考え、それを再帰下向きで構文解析してみることにします。

### 2. 四則演算を含む数式の文法

四則演算を表現する文法を(正規表現を使用して)定義します。
プログラミング言語や数式といった形式言語の文法を表現するためにここでは生成規則を使用しますが、BNF(Backus-Naur Form)でも同様の表記ができます。
はじめの一歩をできるだけ単純にするために、解析対象の数式としてまずは以下の前提をおくことにします。

- operatorは二項演算子 '+', '-', '\*', '/'
- operandは一桁の数字あるいは括弧で囲まれた四則演算

```
E -> E+T | E-T | T
T -> T*F | T/F | F
F -> (E) | N
N -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

これでEを開始記号とし、`5 * (1 + 2 - 10 / 2) + 7`といった数式を扱うことができます。

再帰的下向き構文解析では生成規則から演算子の優先順位が決定されます。
ここではEからTが生成されるため、'+', '-' 演算子よりも'\*', '/' 演算子の優先順位が高く定義されています。

### 3. 文法上の左再帰の解決

再帰的下向き構文解析の利点の一つは、生成規則からプログラムを直感的に書きやすいというところだと思います。
例えば上記規則Fを処理する関数を単純化した擬似的コードで書くと以下のようになります。

```
def F():
    if token == '(':
        E()
        check token == ')'
    else:
        N()
```

実装に規則が反映されており書きやすく読みやすいのですが、定義した文法によっては左再帰の問題が生じることがあります。例えば上記の規則Eを単純にプログラムに落とし込むと以下のようになります。

```
def E():
    E()
    operator()
    T()
```

これだと関数Eの無限呼び出しになってしまいプログラムが終了しません。

調べたところ左再帰を許容する下向き解析のアルゴリズムも存在するよう([Top-down Parsing - Wikipedia][1])なのですが、ここでは生成規則を書き換えることで対応したいと思います。一般的に左再帰が生じる規則は機械的にそれを防ぐように書き換えることができ、上記の規則 `E -> E+T | E-T | T` であれば、下のようになります。

```
E  -> TE'
E' -> +TE' | -TE' | e
('e' means empty)
```

全体としては以下のようになります。

```
E  -> TE'
E' -> +TE' | -TE' | e
T  -> FT'
T' -> *FT' | /FT' | e
F -> (E) | N
N  -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
('e' means empty)
```

### 4. LL(1)文法

これで四則演算を行う数式の文法が定義できたのですが、パーサを実装する前に定義した文法が想定通りLL(1)なのかを確認したいと思います。LL(1)であれば一つのtokenの先読みで、しかもbacktracking無しに解析を行うことができるため、実装が簡単になります。

考え方としてはあるtoken 'x' が与えられた際に、どの生成規則で解析を進めればよいかが常に判断できればよいということになります。
例えば下のような生成規則を考えます。

```
A -> B | C
B -> αB'
C -> βC'
```

ここで \\( \\alpha \\) , \\( \\beta \\) は終端記号、A, B, C, B', C'は非終端記号としています。終端記号、非終端記号といった用語の詳細は[形式文法][6]を参照になるのですが、イメージとしては終端記号が実際の文字列、非終端記号は更に生成規則を適用することで最終的に終端記号が生成されるものという感じになるかと思います。

上の規則の場合、次のtokenが \\( \\alpha \\) であればBの処理を行えばよく、 \\( \\beta \\) であればCの処理を行えばよいことがわかるため、LL(1)的に処理を進めることができます。

一方で

```
A -> B | C
B -> αB'
C -> αC'
```

というような生成規則になっていた場合、token \\( \\alpha \\) を読むだけでは次の生成規則がBなのかCなのかを判断することができず、このような文法はLL(1)とはなりません。

一般にはある生成規則 `A -> B1 | B2 | ... | Bn` を考えた場合に、 \\( B_i (i \\in \\{ 1, 2, .. n \\} ) \\) の先頭に存在し得る終端記号の集合を `First(Bn)` とすると、

\\( First(B_1) \\cap First(B_2) \\cap ... \\cap First(B_n) = \\varnothing \\)

となる必要があるということになります。LL(1)文法ならば、すべての生成規則について上の性質が満たされています。

しかし逆に上の性質が満たされていればLL(1)なのかというとそういうわけではありません。

```
A -> αBc
B -> β(c | e)
('e' means empty)
```

上の例では、「Bで \\( \\beta, c \\) を消費する」場合と、「Bで \\( \\beta \\) を消費する」場合が、先読み1tokenでは判断できません。これはBが空文字を受け入れる場合に、Bで消費する先頭文字とBの後続の文字が一致すると判断に困るということであり、Bの後続の記号集合Followを考えてあげる必要がでてきます。

実際にLL(1)文法であることは、各生成規則に対してFirst集合、Follow集合を求め、そこからDirector集合と呼ばれるものを計算することで確認することができます。First, Follow, Directorの具体的な求め方は[こちら][5]を参照としますが、今回対象としている数式の文法については以下のようになります。

```
生成規則(再掲):
E  -> TE'
E' -> +TE' | -TE' | e
T  -> FT'
T' -> *FT' | /FT' | e
F -> (E) | N
N  -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
('e' means empty)
```

```
First集合:
First(E)  = { 0, 1, .., 9, ( }
First(E') = { +, -, e }
First(T)  = { 0, 1, .., 9, ( }
First(T') = { x, /, e }
First(F)  = { 0, 1, .., 9, ( }
First(N)  = { 0, 1, .., 9 }

Follow集合('$'は入力末尾を表す):
Follow(E)  = { $, ) }
Follow(E') = { $, ) }
Follow(T)  = { +, -, $, ) }
Follow(T') = { +, -, $, ) }
Follow(F)  = { x, /, +, -, $, ) }
Follow(N)  = { x, /, +, -, $, ) }

Director(E, TE')   = First(T) = { 0, 1, .., 9, ( }
Director(E', +TE') = { + }
Director(E', -TE') = { - }
Director(E', e)    = Follow(E') = { $, ) }
Director(T, FT')   = First(F) = { 0, 1, .., 9, ( }
Director(T', xFT') = { x }
Director(T', /FT') = { / }
Director(T', e)    = Follow(T') = { +, -, $, ) }
Director(F, N)     = First(N) = { 0, 1, .., 9 }
Director(F, (E))   = { ( }
(Director(N, _)は明らかなので省略)
```

複数個のDirectorを持つ非終端記号は E', T', Fですが、これらの積が空集合となるため、今回定義した文法はLL(1)であるということができます。

### 5. 構文解析器の実装

構文解析器を実装する準備が整ったので実装に進めます。
なお実装にはPythonを使用しており、コードの全体は[githubリポジトリ][11]に配置しています。

対象とする数式は幾分シンプルなものなので与えられた数式を直接計算結果の数値に変換してもよいのですが、応用の利きやすさから今回は一旦構文木に変換し、それを評価することで計算を行うことにしたいと思います。

解析のためには以下の3つのクラスを用意しました。

- 構文木を表すクラス (expression.py)
- 字句解析し、tokenを作成するクラス (calctokenizer.py)
- 再帰的下向き構文解析を行うクラス (recur_descent.py)

構文木を表すクラスとして数値、二項演算子それぞれに`NumExpr`、`TwoOpExpr`を用意します。
特に変わった実装をしているわけではないためクラスの抜粋のみ記載します。

```python
class NumExpr(Expr):
    def __init__(self, val):
        super().__init__()
        self.val = val

    def evaluate(self):
        return self.val

class TwoOpExpr(Expr):
    def __init__(self, op, left, right):
        super().__init__()
        self.op = op
        self.left = left
        self.right = right

    def evaluate(self):
        return self.op(self.left.evaluate(), self.right.evaluate())
```

字句解析を行うクラスとして`CalcTokenizer`を用意します。
現在の用途だとスペースを飛ばして1文字ずつ読めばよいだけなのでその形で実装していますが、最終的なコードでは任意の(Pythonで表現できる)整数値や2文字以上のtokenを処理するように修正しています。

構文解析器からは`next_token`でtokenを取得できるようにしています。

```python
class CalcTokenizer:
    def __init__(self, source):
        '''
        source: string to be parsed
        '''
        assert isinstance(source, str)
        self.source = source
        self.src_ind = 0
        self.token = None

    def next_char(self):
        '''
        Read a character from the source.
        Return None if the reader reach the end of input.
        '''
        if self.empty():
            return None
        ch = self.source[self.src_ind]
        self.src_ind += 1
        return ch

    def next_token(self):
        '''
        Read a token from the source.
        '''
        self.token = self.next_char()
        if self.token is None:
            return None
        while self.token == ' ':
            self.token = self.next_char()
        return self.token
```

構文解析器(`RecursiveDexcentParser`)は全体を載せると少し長いので、解析の開始から`E -> TE'`、`E' -> +TE' | -TE' | e`の解析を行うところまでを抜粋しています。

開始記号はEなので`parse`関数ではそれに相当する`expr1`を呼び出して解析を開始しています。
各非終端記号に相当する関数の実装は生成規則をそのまま反映したものになっていることがわかると思います。

```python
class RecursiveDescentParser:
    def __init__(self, expression):
        self.tokenizer = CalcTokenizer(expression)

    def parse(self):
      '''
      Parse a given input to the abstract syntax tree.
      '''
      self.tokenizer.next_token()
      return self.expr1()

    def expr1(self):
        '''
        E -> TE'
        '''
        e = self.term1()
        return self.expr2(e)

    def expr2(self, e1):
        '''
        E' -> +TE' | -TE' | e
        '''
        if self.tokenizer.token == '+':
            self.tokenizer.next_token()
            e2 = self.term1()
            return self.expr2(TwoOpExpr(plus, e1, e2))
        elif self.tokenizer.token == '-':
            self.tokenizer.next_token()
            e2 = self.term1()
            return self.expr2(TwoOpExpr(minus, e1, e2))
        else:
            return e1
```

これで簡単な数式の評価が行えるようになりました。

```
> tdp = RecursiveDescentParser('(1 + 2) * 3')
> ast = tdp.parse()
> print(ast)
TwoOpExpr {op=*, left=TwoOpExpr {op=+, left=NumExpr {val=1}, right=NumExpr {val=2}}, right=NumExpr {val=3}}
> print(inp, '=', ast.evaluate())
(1 + 2) * 3 = 9
```

### 6. 単項演算子の追加

前項の数式を少し発展させて'+', '-'といった単項演算子を処理できるようにしたいと思います。これにより例えば`-5 * 2 + 9 / (+3)`といった数式が処理できるようになります。

まずは生成規則にこの変更を反映させます。
単項演算子は'+', '-', '\*', '/'といった二項演算よりも優先順位が高く、数値の前に付けられるものであるため、生成規則Fを以下のように修正します。

```
F -> (E) | N
N -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

↓

```
F -> (E) | +N | -N | N
N -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

続いてコードにも変更を反映させます。
上記生成規則Fに対応するメソッド`factor2`にtokenが'+'、'-'であった場合の処理を追加し、その場合`UnOpExpr`という単項演算子用のexpressionを適用することにします。`UnOpExpr`には渡された数値に-1をかけて返すラムダ式`neg`を渡しています。

```python
def factor2(self):
    if self.tokenizer.is_digit(self.tokenizer.token[0]):
        return self.number()
    elif self.tokenizer.token == '+':
        # this should be like '+53'.
        # skip and parse positive number.
        self.tokenizer.next_token()
        return self.number()
    elif self.tokenizer.token == '-':
        self.tokenizer.next_token()
        return UnOpExpr(neg, self.number())
    elif self.tokenizer.token == '(':
        self.tokenizer.next_token()
        e = self.expr1()
        self.tokenizer.next_token()
        return e
```

これで単項演算子の評価が行えるようになりました。

```
> tdp = RecursiveDescentParser('15*(+3)-202+99/-11+0')
> ast = tdp.parse()
> print(inp, '=', ast.evaluate())
15*(+3)-202+99/-11+0 = -166.0
```

### 7. 右結合演算子の追加

これまであまり触れなかったのですが、二項演算子については優先順位の他に「左結合」か「右結合」かを考える必要があります。

これまで導入した'+', '-', '\*', '/'といった演算子は全て左結合性演算子でした。左結合であるとは`4 + 3 + 2 = (4 + 3) + 2 = 9`のように連続した演算子の評価が左から行われるということになります。

一方でPython等で'\*\*'が表すべき乗は右結合性演算子であり、`4 ** 3 ** 2 = 4 ** (3 ** 2) = 262144`のように評価されます。

このような演算子の結合性も生成規則上で定義することができます。ここでは簡単な例として`A -> A <rop> B | B`という`<rop>`二項演算子の生成規則を考えてみます。先述のように'+'等ではここから生成規則を2つに分けて書き換えました。

ここで今度は生成規則を`A -> B <rop> A | B`のように書き換えます。一見した見た目は違いますが、左再帰性を解決させつつ同一の構文を表すものになっています。`a <rop1> b <rop2> c`という入力(a, b, cはそれぞれBで受け入れられる終端記号、`<rop1>`, `<rop2>`は同一の`<rop>`終端記号だが、位置の違いを明確にするため別名をふっている)に対する解析の流れを考えると、

```
a <rop1> b <rop2> c -> B <rop1> b <rop2> c -> B <rop1> A -> A
```

のようになります。この場合、`<rop1>`よりも`<rop2>`の評価が先に行われており、演算子の結合性としては右結合ということになります。

以上を踏まえ、これまで構築した数式用の生成規則に上述のべき乗を追加した場合、生成規則は以下のように書き換えることができます。

```
E  -> TE'
E' -> +TE' | -TE' | e
T  -> FT'
T' -> *FT' | /FT' | e
F  -> F' | F'**F
F' -> (E) | +N | -N | N
N  -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
('e' means empty)
```

あとはこの生成規則を基に実装にも手を加えれば、べき乗を右結合で正しく処理した計算を行うことができるようになります。

```
> tdp = RecursiveDescentParser('4 ** 3 ** 2')
> ast = tdp.parse()
> print(ast)
TwoOpExpr {op=**, left=NumExpr {val=4}, right=TwoOpExpr {op=**, left=NumExpr {val=3}, right=NumExpr {val=2}}}
> print(inp, '=', ast.evaluate())
4 ** 3 ** 2 = 262144
```

### 8. Summary

- LL(1)文法の解析であれば、再帰的下向き構文解析で比較的簡単に書くことができる
- 演算子文法を扱う場合、優先順位と結合性に気をつける必要がある

後者に関してですが、本記事の通り単純な実装を進めると、演算子が増えるにつれて実装がごちゃごちゃしてくるかもしれません。その場合、数式の評価を[Shunting-yard algorithm][9], あるいは[precedence climbing][10]に任せた構文解析を考える方が実用的になると思います ([参照][3])。

---

### 参考

- [コンパイラ (中田育男氏の本)][7]
- [形式文法][6]
- [Parsing Expressions by Recursive Descent][4]
- [Recursive descent, LL and predictive parsers][2]

[1]: https://en.wikipedia.org/wiki/Top-down_parsing
[2]: http://eli.thegreenplace.net/2008/09/26/recursive-descent-ll-and-predictive-parsers/
[3]: http://www.antlr.org/papers/Clarke-expr-parsing-1986.pdf
[4]: http://www.engr.mun.ca/~theo/Misc/exp_parsing.htm
[5]: http://cis.k.hosei.ac.jp/~asasaki/lect/compiler/2009A/Ex07-2.pdf
[6]: https://ja.wikipedia.org/wiki/%E5%BD%A2%E5%BC%8F%E6%96%87%E6%B3%95 "形式文法"
[7]: https://www.amazon.co.jp/%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%A9-%E6%96%B0%E3%82%B3%E3%83%B3%E3%83%94%E3%83%A5%E3%83%BC%E3%82%BF%E3%82%B5%E3%82%A4%E3%82%A8%E3%83%B3%E3%82%B9%E8%AC%9B%E5%BA%A7-%E4%B8%AD%E7%94%B0-%E8%82%B2%E7%94%B7/dp/4274130134 "コンパイラ"
[8]: https://ja.wikipedia.org/wiki/%E6%96%87%E8%84%88%E8%87%AA%E7%94%B1%E6%96%87%E6%B3%95 "文脈自由文法"
[9]: https://en.wikipedia.org/wiki/Shunting-yard_algorithm "Shunting-yard algorithm"
[10]: http://eli.thegreenplace.net/2012/08/02/parsing-expressions-by-precedence-climbing "Precedence climbing"
[11]: https://github.com/tiqwab/example/tree/master/calc-parser "calc-parser"
