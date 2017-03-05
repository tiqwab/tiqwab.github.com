---
layout: post
title: 数式の構文解析 with PLY
tags: "lex, yacc, ply, python"
comments: true
---

以前 [数式の再帰的下向き構文解析器][2] を手書きで書いたのですが、それと同じ文法に対する構文解析器を 今度は [PLY][1] を使用して書きました。
元々 LALR parser の理解という目的もあって lex, yacc を使ってみたいと思っていたのですが、 C で実装するのが少し面倒だなと思っていたので、(特に実用目的でもないですし) 手軽に扱える PLY を使用してみています。

1. [ply とは](#anchor1)
2. [ply.lex による lexer の生成](#anchor2)
3. [ply.yacc による parser の生成](#anchor3)

### 環境

- OS: CentOS Linux release 7.3.1611
- Python: 3.6.0
- PLY: 3.9

---

<a id="anchor1"></a>

### 1. ply とは

- [lex][7], [yacc][4] の Python 実装
- lex では各 token の正規表現を与えて lexer (字句解析器) を生成する
- yacc では LALR(k) 文法に対する parser (構文解析器) の生成が可能
  - ただし通常は LALR(1) 文法に使用される

<a id="anchor2"></a>

### 2. ply.lex による lexer の生成

`ply.lex` の提供する機能を利用して、以前 [数式の再帰的下向き構文解析][2] で使用した文法に対する lexer を作成しました (code: [lexer for calculator][5])。

```
Grammars for math expression:
E  -> TE'
E' -> +TE' | -TE' | e
T  -> FT'
T' -> *FT' | /FT' | e
F  -> F' | F'**F
F' -> (E) | +N | -N | ab F' | N
N  -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
('e' means empty)
('ab' means absolute value)
```

この文法では例えば `(-3 + 2 * 5 ** 2) * 6 / 2 - (ab (-10))` のような数式を生成することができます。

<a id="anchor3"></a>

### 3. ply.yacc による parser の生成

同様に `ply.yacc` の提供する機能を利用して、上記文法で生成される数式に対する parser を作成しました (code: [parser for calculator][6])。

上の文法は下向き構文解析を書きやすくするために多少定義に工夫が必要だったのですが、それと比べると yacc の場合は演算子の associativity や precedence を別途宣言すれば凄い適当な文法定義でも何とかなります。

```
Grammars used to generate a parser by yacc:
S -> E
E -> E + E
E -> E - E
E -> E * E
E -> E / E
E -> E ** E
E -> N
E -> +N
E -> -N
E -> ab E
E -> (E)
```

### 参考

- [PLY (Python Lex-Yacc][1]
- [LALR parser][3]

[1]: http://www.dabeaz.com/ply/ "PLY"
[2]: {% post_url 2017-01-04-recursive-descent-parser %} "数式の再帰的下向き構文解析"
[3]: https://en.wikipedia.org/wiki/LALR_parser "LALR parser"
[4]: https://ja.wikipedia.org/wiki/Yacc "Yacc"
[5]: https://github.com/tiqwab/example/blob/master/calc-parser/ply/calclexer.py "calc-lexer"
[6]: https://github.com/tiqwab/example/blob/master/calc-parser/ply/calcparser.py "calc-parser"
[7]: https://ja.wikipedia.org/wiki/Lex "Lex"
