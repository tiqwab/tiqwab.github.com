---
layout: post
title: 命題論理について
tags: "propositional logic"
comments: true
---

『[論理と計算のしくみ][1]』を参考に。

---

### 構文論

命題論理の構文論 (syntax) では命題論理式 (propositional formula)、あるいは単に論理式 (formula) と呼ばれるものを定義する。
論理式は命題記号、真偽値を表す記号、論理記号から帰納的に定義される。

- 命題記号 (propositional symbol)
  - 命題を表す記号で \\( P \\) , \\( Q \\) のように表される
  - 原子論理式 (atomic formula) とも呼ばれる
- 真偽値を表す記号
  - ここでは \\( T \\) , \\( F \\) とする
- 論理記号 (logical symbol)
  - 論理式に付加する記号
  - \\( \\lnot \\) , \\( \\land \\) , \\( \\lor \\) , \\( \\Rightarrow \\) といったもの

例えば \\( (P \\land Q) \\lor ((\\lnot P \\land Q) \\lor (\\lnot Q \\land T)) \\) は論理式である。

構文論ではあくまで論理式がどのように表記されるのか、を定義しているだけであり、その意味 (真偽) については扱わないという点に注意する。

命題論理の構文は生成規則で以下のように表現することができる。

```
S  -> R
R  -> NR'
R' -> or NR' | e
N  -> PN'
N' -> and PN' | e
P  -> T | F | A | not P | ( R )

('or' : disjunction 
 'and': conjunction
 'not': negation
 'T'  : true
 'F'  : false
 'A'  : arbitrary propositional symbol
)
```

### 意味論

命題論理の意味論 (semantics) では論理式の真偽がどのように決定されるかを定義する。

そのためには命題記号の解釈 (interpretation) を定義する必要がある。解釈とは命題記号に真偽値を対応させる関数であり、 \\( I: \\mathbb{P} \\rightarrow \\mathbb{B} \\) と表される。このとき \\( \\mathbb{P} \\) は命題記号の集合、 \\( \\mathbb{B} \\) は \\( \\{ T , F \\} \\) の集合である。

論理式の真偽値は構文論の定義と同様に帰納的に定義されており、解釈を与えることで決定される。

命題論理の構文、意味論は以下のような抽象木で表すことができる。

```python

'''
Abstract Syntax Tree for propositional logic.
'''

class PropositionS(namedtuple('PropositionS', ['symbol'])):
    __slots__ = ()

    def interpret(self, interpreter):
        return interpreter.interpret_proposition(self.symbol)

class TrueS(namedtuple('TrueS', [])):
    __slots__ = ()

    def interpret(self, interpreter):
        return True

class FalseS(namedtuple('FalseS', [])):
    __slots__ = ()

    def interpret(self, interpreter):
        return False

class NotS(namedtuple('NotS', ['formula'])):
    __slots__ = ()

    def interpret(self, interpreter):
        return not self.formula.interpret(interpreter)

class OrS(namedtuple('OrS', ['left', 'right'])):
    __slots__ = ()

    def interpret(self, interpreter):
        return self.left.interpret(interpreter) or self.right.interpret(interpreter)

class AndS(namedtuple('AndS', ['left', 'right'])):
    __slots__ = ()

    def interpret(self, interpreter):
        return self.left.interpret(interpreter) and self.right.interpret(interpreter)

'''
Simulator of interpretation.
'''

class Interpreter:
    def __init__(self, interpretation):
        self.interpretation = interpretation

    def interpret(self, formula):
        return formula.interpret(self)

    def interpret_proposition(self, symbol):
        return self.interpretation[symbol]

def interpret_formula(source, interpretation):
    tokenizer = Tokenizer(source)
    parser = Parser(tokenizer)
    ast = parser.parse()
    interpreter = Interpreter(interpretation)
    return interpreter.interpret(ast)

if __name__ == '__main__':
    assert interpret_formula('not P and Q or Q', {'P': True, 'Q': False}) == False
    assert interpret_formula('not P and Q or Q', {'P': False, 'Q': True}) == True
```

### 恒真

形式論理において、任意の解釈で真となる論理式のことを恒真 (valid) であるという。特に命題論理では恒真な論理式をトートロジ (tautology) ともいう。

トートロジとしては例えば以下の2つがある。 各論理式は \\( A \\) を真偽どちらに解釈しても論理式全体では真となるため、トートロジである。

- 排中律 ( \\( A \\lor \\lnot A \\) )
- 二重否定の除去 ( \\( \\lnot \\lnot A \\Rightarrow A \\) )

ある論理式に現れる命題の数は有限なので、その論理式がトートロジかどうかは決定可能である。つまり命題論理においては任意の論理式をトートロジか有限時間内に判定するアルゴリズムが存在する。

### 充足可能と充足不能

論理式 \\( A \\) に対し、充足可能 (satisfiable)、充足不能 (unsatisfiable) は以下のように定義される。

- ある解釈 \\( I \\) で \\( A \\) を真とできるならば、 \\( A \\) は充足可能であるという
- 任意の解釈で \\( \\lnot A \\) が真ならば、 \\( A \\) は充足不能であるという

### コンパクト性

『論理と計算のしくみ』では形式論理におけるコンパクト性 (compactness) を以下のように表現している。

> 論理式の集合に対して、その任意の有限部分集合が個別に充足可能ならば、集合全体を同時に充足する解釈が存在する

命題論理式の集合 \\( \\varGamma \\) が充足可能であるとは、ある解釈 \\( I \\) が存在し、任意の命題論理式 \\( A \\in \\varGamma \\) に対して \\( I(A) = T \\) が成立することである。このとき \\( \\varGamma \\) は非可算無限でもよい。

よって命題論理におけるコンパクト性は、 \\( \\varGamma \\) の任意の有限部分集合 \\( \\varDelta \\subseteq \\varGamma \\) が個別に充足可能ならば、 \\( \\varGamma \\) を充足する解釈が存在するということを述べている。

証明は専門書を参照。『論理と計算のしくみ』では極大無矛盾集合を利用した証明を載せている。

### 演繹体系

公理 (axiom) と呼ばれる論理式と有限回の推論規則 (inference rule) の適用から定理 (theorem) と呼ばれる論理式を導出する形式的体系を演繹体系 (deduction system) と呼ぶ。ざっくりとした表現をすれば、人間の推論の過程を形式化しよう、というのが演繹体系の目的だといえる。

特に命題論理の演繹体系では、トートロジを定理として導出することを目指す。

#### 健全性と完全性

演繹体系に期待される性質として、健全性と完全性がある。

- 任意の定理がトートロジである場合、その演繹体系が健全 (soundness) であるという
- 任意のトートロジが定理として導ける場合、その演繹体系が完全 (completeness) であるという

命題論理式の集合 \\( \\varGamma \\) から論理式 \\( A \\) が定理として導ける (証明可能) ということを \\( \\varGamma \\vdash A \\) と表記する。
また \\( \\varGamma \\) を充足する解釈 \\( I \\) が 論理式 \\( A \\) も充足するということを \\( \\varGamma \\models A \\) と表記する。

この場合、以下の式の左から右が健全性、右から左が完全性を表す。

\\( \\varGamma \\vdash A \\iff \\varGamma \\models A \\)

健全性は形式的体系として必須ともいえるものだが、完全性は常に実現されるとは限らない。
命題論理における演繹体系はいずれも両者を備えている。

#### 演繹体系の例

命題論理の演繹体系としては以下のようなものがある。

- ヒルベルト流
- 自然演繹
- シーケント計算
- 導出原理

[1]: https://www.amazon.co.jp/%E8%AB%96%E7%90%86%E3%81%A8%E8%A8%88%E7%AE%97%E3%81%AE%E3%81%97%E3%81%8F%E3%81%BF-%E5%B2%A9%E6%B3%A2%E3%82%AA%E3%83%B3%E3%83%87%E3%83%9E%E3%83%B3%E3%83%89%E3%83%96%E3%83%83%E3%82%AF%E3%82%B9-%E8%90%A9%E8%B0%B7-%E6%98%8C%E5%B7%B1/dp/4007305803/ref=dp_ob_title_bk
