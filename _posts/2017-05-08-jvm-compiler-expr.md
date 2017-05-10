---
layout: post
title: 数式のコンパイル for JVM
tags: "jvm, javacc, asm"
comments: true
---

簡単な JVM 言語を作成するための第一歩として、整数式を入力とし、その計算を行う Java bytecode を出力するコンパイラを作成してみたいと思います。

具体的にいえば `(2 + 3) * 4` を受け取ると、何か class ファイルを生成し、それを `java <class>` で実行すると 整数計算を行い 20 が出力される、というような単純なものです。

コンパイルの流れとしては

```
数式
↓ 字句解析、構文解析 by JavaCC
構文木
↓ Java bytecode 出力 by ASM
class ファイル
```

という感じで、字句解析と構文解析に [JavaCC][7]、Java bytecode の生成に [ASM][8] を使用します。

使用しているプログラムの全体は [jvm-compiler-expr][9] に置いています。

1. [パーサ作成 with JavaCC](#javacc)
2. [bytecode 生成 with ASM](#asm)

### 環境

- JVM: openjdk version "1.8.0\_121"

---

<div id="javacc" />

### 1. パーサ作成 with JavaCC

JavaCC は jj (or jjt) ファイルで字句規則と構文規則を定義し、LL(k) パーサを生成します。
JavaCC の概要に関しては[以前の post][3] を参考とし、ここでは主に数式のパースを行うための jjt ファイル定義をさらいます。

#### 生成規則の確認

パーサ作成の前に、まずは目的とする数式の生成規則を確認しておきます。
ここでは JavaCC の求める形に近い[EBNF][1] で構文を表現しています。

```
S ::= E
E ::= T ("+" T | "-" T)*
T ::= F ("*" T | "/" F)*
F ::= N | "(" E ")"
```

この規則に基づいて jjt ファイルを作成します。

#### options の定義

JavaCC の生成するパーサに関する設定を `options` 内に定義します。
各設定に関しては[こちら][2]にいくつか解説があります。

細かいところまではおさえていませんが、とりあえずは以下が無難な設定に思います。

```
options {
    STATIC = false;
    JAVA_UNICODE_ESCAPE = true;
    UNICODE_INPUT = true;
}
```

#### パーサクラスの定義

生成されるパーサのもとになるクラスを `PARSER_BEGIN`, `PARSER_END` 内に定義します。

ここではパーサの package, class 名を定義し、また処理中に使用する `JBCNode` クラスの import をしておきます (後述)。

```
PARSER_BEGIN (Parser)

package com.tiqwab.example;

import com.tiqwab.example.jbc.*;

public class Parser {
}
PARSER_END (Parser)
```

#### 字句の定義

数式を表現するために必要な字句を定義します。
`TOKEN` が構文解析に使用する各字句の定義、`SKIP` が読み飛ばす字句の定義になります。

各字句は正規表現で定義できるのですが、最終的に Java ファイルとなるためか文字の表現が少し冗長なものになります (e.g. NUMBER の定義を `[0-9]+` ではなく `["0"-"9"]+` のように書く必要がある)。

```
// White space
SKIP: {
    " "
    | "\t"
    | "\r"
    | "\f"
}

TOKEN: {
    < NEWLINE: "\n" >
}

// Operator
TOKEN: {
    < ADD: "+" >
    | < SUBTRACT: "-" >
    | < MULT: "*" >
    | < DIV: "/" >
}

// Symbols
TOKEN: {
    < LPAREN: "(" >
    | <RPAREN: ")" >
}

// Number
TOKEN: {
    < NUMBER: (["0"-"9"])+ >
}
```

#### 構文の定義

最後に数式の構文規則を定義します。

非終端記号名と jjt ファイル内の関数名を一致させた EBNF を再掲します。

```
<Start>      ::= <Expression>
<Expression> ::= <Term> ("+" <Term> | "-" <Term>)*
<Term>       ::= <Factor> ("*" <Term> | "/" <Factor>)*
<Factor>     ::= int | "(" <Expression> ")"
```

jjt ファイル内での構文定義では、非終端記号のパースが完了したタイミングで独自に定義した `JBCNode` を生成し、構文木を作成するようにしています。
ここで生成している木は `JBCBinaryOperator` が節、 `JBCInteter` が葉となる二分木です。
なお、木を構成するクラスはいずれも `JBCNode` インタフェースを実装したものとしています (後述)。

実は JavaCC 単体でも構文規則に基づいて木構造を自動で生成することはできるのですが、そうして生成される木よりも operator を節とする二分木の方が Java bytecode 生成処理を行いやすいため新たに木構造を定義しています。

```
// <Start>      ::= <Expression>
JBCNode Start(): {
    JBCNode expression;
} {
    expression = Expression() <NEWLINE> {
        return new JBCEval((JBCExpr) expression);
    }
}

// <Expression> ::= <Term> ("+" <Term> | "-" <Term>)*
JBCNode Expression(): {
    JBCNode term1;
    JBCNode term2;
    JBCNode expr;
} {
    term1 = Term() { expr = term1; } (
        <ADD> term2 = Term() {
            expr = new JBCBinaryOperator("+", expr, term2);
        }
        | <SUBTRACT> term2 = Term() {
            expr = new JBCBinaryOperator("-", expr, term2);
        }
    )* {
        return expr;
    }
}

// <Term>       ::= <Factor> ("*" <Term> | "/" <Factor>)*
JBCNode Term(): {
    JBCNode factor1;
    JBCNode factor2;
    JBCNode term;
} {
    factor1 = Factor() { term = factor1; } ( 
        <MULT> factor2 = Factor() {
            term = new JBCBinaryOperator("*", term, factor2);
        }
        | <DIV> factor2 = Factor() {
            term = new JBCBinaryOperator("/", term, factor2);
        }
    )* {
        return term;
    }
}

// <Factor>     ::= int | "(" <Expression> ")"
JBCNode Factor(): {
    Token number;
    JBCNode expr;
} {
    number = <NUMBER> {
        return new JBCInteger(number.image);
    }
    | <LPAREN> expr = Expression() <RPAREN> {
        return expr;
    }
}
```

この jjt ファイルを JavaCC に与えて生成される `Parser` クラスを使用すると、簡単に標準入力から数式文字列を受け取り、`JBCNode` で構成される二分木を生成するプログラムを書くことができます。

```java
public static void main(String[] args) throws Exception {
    Parser parser = new Parser(System.in); // 標準入力から数式の受け取り
    JBCNode node = parser.Start(); // 構文木の生成
}
```

<div id="asm" />

### 2. bytecode 生成 with ASM

ここまでの実装で入力された数式から構文木を作成するところまでできたので、次にこの木構造を利用して Java bytecode を生成する処理を行います。

#### JBCNode と JBCNodeVisitor 定義

構文木の処理には Visitor パターンを利用したいので、`JBCNode`、`JBCNodeVisitor` インタフェースを定義します。
`JBCNode` で構成される木構造に `JBCNodeVisitor` で処理を行わせます。

```java
public interface JBCNode {

    public void accept(JBCNodeVisitor visitor);

}
```

```java
public interface JBCNodeVisitor {

    public void visit(JBCEval node);
    public void visit(JBCBinaryOperator node);
    public void visit(JBCInteger node);

}
```

`JBCNode` の実装クラスとして `JBCInteger`, `JBCBinaryOperator`, `JBCEval` を用意します (実際は今後のことを考え、`JBCNode` を実装した `JBCStmt`, `JBCExpr` インタフェースを用意し、各 Node にはこれを実装させているが、今回の範囲では関係することがないので省略)。

```java
public class JBCInteger implements JBCExpr {

    private final int value;

    public JBCInteger(final String value) {
        this.value = Integer.parseInt(value);
    }

    public int getValue() {
        return this.value;
    }

    @Override
    public String toString() {
        return String.format("JBCInteger{value=%s}", value);
    }

    @Override
    public void accept(JBCNodeVisitor visitor) {
        visitor.visit(this);
    }

}
```

```java
public class JBCBinaryOperator implements JBCExpr {

    private final String op;
    private final JBCNode lhs;
    private final JBCNode rhs;

    public JBCBinaryOperator(final String op, final JBCNode lhs, final JBCNode rhs) {
        this.op = op;
        this.lhs = lhs;
        this.rhs = rhs;
    }

    public String getOp() {
        return this.op;
    }

    @Override
    public String toString() {
        return String.format("JBCBinaryOperator{op=%s, lhs=%s, rhs=%s}", this.op, this.lhs, this.rhs);
    }

    @Override
    public void accept(JBCNodeVisitor visitor) {
        lhs.accept(visitor);
        rhs.accept(visitor);
        visitor.visit(this);
    }

}
```

```java
public class JBCEval implements JBCStmt {

    private final JBCExpr expr;

    public JBCEval(final JBCExpr expr) {
        this.expr = expr;
    }

    @Override
    public String toString() {
        return String.format("JBCEval{expr=%s}", this.expr.toString());
    }

    @Override
    public void accept(JBCNodeVisitor visitor) {
        expr.accept(visitor);
        visitor.visit(this);
    }

}

```

例えば `(1 + 2) * 3` に対しては以下のような構文木が生成されます。

```
                                   JBCEval
                                     | 
                         JBCBinaryOperator{op="*"}
                               /           \
        JBCBinaryOperator{op="+"}         JBCInteger{value=3}
           /               \
JBCInteger{value=1} JBCInteger{value=2}
```

(`JBCEval` も今回の処理の範囲では必要ないので実際は省略可)

#### bytecode を生成する Visitor 定義

Java bytecode を生成する処理は `JBCNodeVisior` を実装した `JBCGenerationVisitor` に行ってもらいます。

ここで `JBCGenerationVisitor` に求められる処理は以下のものです。

1. 数式計算を実行する `Calculator` クラスの定義
2. コンストラクタとプログラム実行用 `main` メソッドの定義
3. 構文木に基づいた数式計算用 bytecode の生成

このうち 1. と 2. は数式計算に関係する内容ではなく、プログラムを実行するために最低限必要な内容の定義を行う箇所です。
現在の単純なコンパイラだとクラスやメソッドを作り出せないため、こちらで勝手に生成するようにしています。

最終的に生成したいクラスを Java として書くと以下のようになります。
ここではこれと同様の処理を行う Java bytecode を直接生成することが目標です。

```java
public class Calculator {
    public static void main(String[] args) {
        System.out.println(calculate());
    }
    public static int calculate() {
        ... (calculation of expression)
    }
}
```

#### 1. 数式計算を実行する Calculator クラスの定義

ASM を使用して `Calculator` クラスの定義を行います。 ASM による Java bytecode 生成処理では主に `ClassWriter`, `MethodVisitor` が中心になります。

- (1) ASM が提供する `ClassWriter` で、主にクラスレベルの処理を行う
- (2) ASM で生成した Java bytecode をファイルに出力するために定義した独自クラス
- (3) ASM が提供する `MethodVisitor` で、主にメソッドレベルの処理を行う

```java
import com.tiqwab.example.GeneratedCode;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

public class JBCGenerationVisitor implements JBCNodeVisitor {

    // (1)
    private ClassWriter classWriter;
    private String generatedClassName;
    // (2)
    private GeneratedCode generatedCode;

    // (3)
    private MethodVisitor mv;

    public JBCGenerationVisitor() {
        this.generatedClassName = "Calculation";
    }

    ...
}
```

`ClassWriter` による `Calculator` クラス定義の生成は以下のように行います。

- (1) Java bytecode 生成前の初期化用メソッド
- (2) `ClassWriter` の初期化
  - 渡している引数については[以前の post][4] を参考
- (3) `Calculator` クラス定義の生成

```java
    // (1)
    private void preVisit() {
        // (2)
        this.classWriter = new ClassWriter(ClassWriter.COMPUTE_FRAMES | ClassWriter.COMPUTE_MAXS);
        // (3)
        this.classWriter.visit(
                Opcodes.V1_8,
                Opcodes.ACC_PUBLIC | Opcodes.ACC_SUPER,
                generatedClassName,
                null,
                "java/lang/Object",
                null
        );
    }
```

#### 2. コンストラクタとプログラム実行用 main メソッドの定義

上で例示した Java のソースファイルではコンストラクタを省略していたのですが、通常コンパイルするとデフォルトコンストラクタが自動生成されるので、ここでもそれに合わせて定義するようにします (ただし今回のように static メソッドのみで処理を行う場合、bytecode 上では必須では無いようで省略できる)。

デフォルトコンストラクタは javap 上では以下のように表示されるので、これと同じことを ASM で行います。

```
  public Calculator();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
```

- (1) メソッド定義を行うために `MethodVisitor` を生成する
- (2) メソッド内部の定義開始
- (3) メソッド内部の定義終了

```java
    private void visitConstructor() {
        // (1)
        MethodVisitor mv = classWriter.visitMethod(
                Opcodes.ACC_PUBLIC,
                "<init>",
                "()V",
                null,
                null
        );
        // (2)
        mv.visitCode();
        mv.visitVarInsn(Opcodes.ALOAD, 0);
        mv.visitMethodInsn(
                Opcodes.INVOKESPECIAL,
                "java/lang/Object",
                "<init>",
                "()V",
                false
        );
        mv.visitInsn(Opcodes.RETURN);
        mv.visitMaxs(0,0);
        // (3)
        mv.visitEnd();
    }

```

次にプログラムのエントリポイントである `main` メソッドを定義します。

こちらも同様に javap で出力される内容に従って ASM で定義していけば大丈夫です。

```
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: invokestatic  #3                  // Method calculate:()I
         6: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
         9: return
      LineNumberTable:
        line 3: 0
        line 4: 9
```

```java
    private void visitMain() {
        MethodVisitor mv = classWriter.visitMethod(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC,
                "main",
                "([Ljava/lang/String;)V",
                null,
                null
        );
        mv.visitCode();

        // Print calculation result
        mv.visitFieldInsn(
                Opcodes.GETSTATIC,
                "java/lang/System",
                "out",
                "Ljava/io/PrintStream;"
        );
        mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                "Calculation",
                "calculate",
                "()I",
                false
        );
        mv.visitMethodInsn(
                Opcodes.INVOKEVIRTUAL,
                "java/io/PrintStream",
                "println",
                "(I)V",
                false
        );
        mv.visitInsn(Opcodes.RETURN);

        mv.visitMaxs(0,0);
        mv.visitEnd();
    }
```

#### 3. 構文木に基づいた数式計算用 bytecode の生成

JavaVM では operand stack を利用して整数の計算を行うため、逆ポーランド記法と同様な順番で opcode を並べていくことになります (JavaVM 処理系についての概要は [Stack on JavaVM][5] を参考)。

これは今回使用している構文木に基づいていえば、各 node を post-order 的に処理していくことになります。
例えば `2 + 3` の場合、

```
1. 整数 2 をスタックに積む
2. 整数 3 をスタックに積む
3. + 演算を行い、その結果 (整数 5) をスタックに積む
```

という順番で 3 つの命令を生成します。

post-order 的に処理を行うことに関しては、各 `JBCNode` 実装クラスの `accept` メソッドで定義済みなので、 Visitor 側では対応する bytecode を生成するようにします。

```java
    @Override
    public void visit(JBCEval node) {

    }

    // Generate code to perform addition, subtraction, multiplication, and division
    @Override
    public void visit(JBCBinaryOperator node) {
        if (node.getOp().equals("+")) {
            mv.visitInsn(Opcodes.IADD);
        } else if (node.getOp().equals("-")) {
            mv.visitInsn(Opcodes.ISUB);
        } else if (node.getOp().equals("*")) {
            mv.visitInsn(Opcodes.IMUL);
        } else if (node.getOp().equals("/")) {
            mv.visitInsn(Opcodes.IDIV);
        } else {
            throw new IllegalArgumentException("unknown op: " + node.getOp());
        }
    }

    // Generate code to load integer to stack
    @Override
    public void visit(JBCInteger node) {
        final int value = node.getValue();
        // The generated code is determined by the necessary size (byte) of integer.
        if (-128 <= value && value < 128) {
            mv.visitIntInsn(Opcodes.BIPUSH, node.getValue());
        } else if (-32768 <= value && value < 32768){
            mv.visitIntInsn(Opcodes.SIPUSH, node.getValue());
        } else {
            mv.visitLdcInsn(value);
        }
    }
```

上と同様の例 (`2 + 3`) の場合、以下の Java bytecode が生成されます。

```
bipush 2
bipush 3
iadd
```

最後に 1, 2, 3 で見てきたメソッドを組み合わせて、構文木から `Calculator` クラスの bytecode を生成するメソッドを定義します。

```java
    // (1)
    public void generateCode(JBCNode node) {
        this.preVisit();
        this.visitConstructor();
        this.visitMain();

        // Start creation of method to calculate
        mv = classWriter.visitMethod(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC,
                "calculate",
                "()I",
                null,
                null
        );
        mv.visitCode();

        // Generate java byte code to calculate expression
        node.accept(this);

        // Finish creation
        mv.visitInsn(Opcodes.IRETURN);
        mv.visitMaxs(0,0);
        mv.visitEnd();

        this.generatedCode = new GeneratedCode(this.classWriter.toByteArray(), this.generatedClassName);
    }
```

`ClassWriter#toByteArray` で `Calculator` クラス定義を表すバイナリが渡されるので、これを適当にカレントディレクトリにでも出力すれば JVM 上で実行できるプログラムが入手できます。

```
$ ./gradlew clean run
(1 + 2) * 5

$ javap -c -v Calculator
public class Calculation
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Utf8               Calculation
   #2 = Class              #1             // Calculation
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = NameAndType        #5:#6          // "<init>":()V
   #8 = Methodref          #4.#7          // java/lang/Object."<init>":()V
   #9 = Utf8               main
  #10 = Utf8               ([Ljava/lang/String;)V
  #11 = Utf8               java/lang/System
  #12 = Class              #11            // java/lang/System
  #13 = Utf8               out
  #14 = Utf8               Ljava/io/PrintStream;
  #15 = NameAndType        #13:#14        // out:Ljava/io/PrintStream;
  #16 = Fieldref           #12.#15        // java/lang/System.out:Ljava/io/PrintStream;
  #17 = Utf8               calculate
  #18 = Utf8               ()I
  #19 = NameAndType        #17:#18        // calculate:()I
  #20 = Methodref          #2.#19         // Calculation.calculate:()I
  #21 = Utf8               java/io/PrintStream
  #22 = Class              #21            // java/io/PrintStream
  #23 = Utf8               println
  #24 = Utf8               (I)V
  #25 = NameAndType        #23:#24        // println:(I)V
  #26 = Methodref          #22.#25        // java/io/PrintStream.println:(I)V
  #27 = Utf8               Code
{
  public Calculation();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #8                  // Method java/lang/Object."<init>":()V
         4: return

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #16                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: invokestatic  #20                 // Method calculate:()I
         6: invokevirtual #26                 // Method java/io/PrintStream.println:(I)V
         9: return

  public static int calculate();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: bipush        1
         2: bipush        2
         4: iadd
         5: bipush        5
         7: imul
         8: ireturn
}

$ java Calculator
15
```

#### 四則演算処理の補足

簡単な整数計算を行う Java bytecode 生成については以上になります。

他に Java VM 上での四則演算処理命令に関してはこれらもケアしておきたいかなと思います。

- 型ごとの四則演算の処理命令 (xADD, xSUB, xMUL, xDIV) ([2.11.3 in jvm specification][6])
  - int, float, double で命令が異なる
  - int の ADD なら iadd, float なら fadd, double なら dadd
- int の定数値の load 命令 (BIPUSH, SIPUSH, LDC)
  - 使用頻度や必要な byte 数に応じて処理命令が異なる
  - -1 から 5 に関しては専用の命令 `iconst_<i>` が用意されている
  - 2 byte に収まる範囲の整数ならば `bipush`
  - それ以外の整数は constant pool に置かれ、`ldc` で操作される

[1]: https://ja.wikipedia.org/wiki/EBNF "EBNF"
[2]: http://cis.k.hosei.ac.jp/~nakata/lectureCompiler/JavaCC/index.html
[3]: {% post_url 2017-04-30-javacc-and-asm %}
[4]: http://127.0.0.1:4000/2017/04/30/javacc-and-asm.html#asm
[5]: https://www.slideshare.net/skrb/stack-on-javavm-4932723
[6]: https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.11.3
[7]: https://javacc.org/
[8]: http://asm.ow2.org/
[9]: https://github.com/tiqwab/example/tree/master/jvm-compiler-expr
