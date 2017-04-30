---
layout: post
title: JavaCC と ASM
tags: "jvm, javacc, asm"
comments: true
---

最近 JVM (Java Virtual Machine) 上で実行できる `.class` ファイル (Java bytecode) を生成する簡単なコンパイラを書いてみたいと思っており、そのために JavaCC と ASM というライブラリの使用方法を確認しています。

これらは初めて使用するライブラリだったのですが、触っていて面白く使いやすいライブラリだと感じたので、使い方について簡単にまとめておこうと思います。

参考にできるかは怪しいですがここでまとめた内容は [tiqwab/simple-jvm-lang][15] を元にしています。

1. [JavaCC](#javacc)
2. [ASM](#asm)

### 環境

- JVM: openjdk version "1.8.0\_121"

---

<div id="javacc" />

### 1. JavaCC

[JavaCC][6] (Java Compiler Compiler) は Java で実装されたパーサジェネレータです。
JavaCC に構文規則を記述した専用のファイル (`.jj` ファイル) を渡すと、対応するパーサが Java ソースファイル (`.java` ファイル) として生成されます。

パーサジェネレータというと [yacc][7] が有名ですが、JavaCC が 対象とするのは LL(k) 文法のみです。
特に通常は LL(1) を認識するパーサを生成します。

ここでは JavaCC の簡単な使い方を含め以下の内容について整理してみます。

- Gradle プラグインを介した JavaCC の実行
- jj ファイルの書き方
- jjtree を利用した解析木の作成

#### Gradle プラグインを介した JavaCC の実行

JavaCC は 公式で javacc, jjtree のようなコマンドを提供していますが、ここでは簡便のために [javaccPlugin][2] という Gradle プラグインを利用することにします。
このプラグインを利用することで、 Gradle のタスクとして JavaCC の処理を行うことができます。

javaccPlugin の README を参照しつつ、`build.gradle` ファイルを以下のように作成しました。

- (1) `plugins` に javaccPlugin を追加する
- (2) javacc, jjtree で生成される Java ファイルの出力先を指定する
  - ここでは `src/main/generated` を指定
- (3) clean 時に javacc, jjtree で生成されたファイルも削除されるようにする
- (4) javacc, jjtree で生成したファイルもビルド時にソースファイルとして扱われるようにする

```groovy
// (1)
plugins {
    id 'java'
    id 'ca.coglinc.javacc' version '2.4.0'
}

group 'com.tiqwab'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
}

// --- Tasks ---

// (2)
compileJjtree {
    outputDirectory = file("${rootProject.projectDir}/src/main/generated")
}

// (2)
compileJavacc {
    outputDirectory = file("${rootProject.projectDir}/src/main/generated")
}

// (3)
task cleanGeneratedFiles(type: Delete) {
    delete file("${rootProject.projectDir}/src//main/generated")
}
clean.dependsOn cleanGeneratedFiles

// (4)
sourceSets {
    main {
        java {
            srcDir compileJjtree.outputDirectory
            srcDir compileJavacc.outputDirectory
        }
    }
}
```

これで `./gradlew clean build` で以下の内容が行われるようになります。

- `src/main/javacc/*.jj` を処理し `src/main/generated` 下にパーサを生成
- `src/main/jjtree/*.jjt` を処理し `src/main/generated` 下にパーサを生成
- ビルド時には `src/main/generated` 下のファイルもコンパイルして使用

コンパイラを作成する際には `src/main/generated` 下に生成されたパーサを利用して構文解析を行うことを想定しています。

#### jj ファイルの書き方

JavaCC では生成するパーサに必要な情報 (字句規則、構文規則 etc.) は `.jj` ファイルに記述します。
例えばただ数字のみを受理するパーサを生成した場合、以下のような `src/main/javacc/Parser.jj` ファイルを用意すれば OK です。

```
// --- Definition of options ---
options {
    STATIC = false;
    JAVA_UNICODE_ESCAPE = true;
    UNICODE_INPUT = true;
}

// --- Definition of parser ---
PARSER_BEGIN (Parser)

package com.tiqwab.example;

public class Parser {
}
PARSER_END (Parser)

// --- Definition of tokens ---

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

// Number
TOKEN: {
    < NUMBER: (["0"-"9"])+ >
}

// --- Translation Scheme ---

void number(): {
    Token number;
} {
    number = <NUMBER> <NEWLINE> {
        System.out.println(number);
    }
}
```

これから生成されるパーサは Java プログラムから簡単に使用することができます。

```java
public class ParserMain {
    public static void main(String[] args) throws Exception {
        Parser parser = new Parser(System.in);
        parser.number();
    }
}
```

```
> java com.tiqwab.example.ParserMain
> 20
20

> java com.tiqwab.example.ParserMain
> foo
Exception in thread "main" com.tiqwab.example.TokenMgrError: Lexical error at line 1, column 1.  Encountered: "f" (102), after : ""
    at com.tiqwab.example.ParserTokenManager.getNextToken(ParserTokenManager.java:411)
    at com.tiqwab.example.Parser.jj_consume_token(Parser.java:133)
    at com.tiqwab.example.Parser.number(Parser.java:10)
    at com.tiqwab.example.ParserMain.main(ParserMain.java:7)
```

`.jj` ファイルは主に 4 つの要素から構成されます。

1. パーサクラスの宣言
2. 字句規則
3. 構文規則
4. オプション

#### 1. パーサクラスの宣言

`.jj` ファイルでは `PARSER_BEGIN`, `PARSER_END` で囲まれた箇所で生成されるパーサクラスの定義を行います。
ここで定義したクラスをもとに JavaCC はパーサを作成します。

```
PARSER_BEGIN (Parser)

package com.tiqwab.example;

public class Parser {
}
PARSER_END (Parser)
```

上記のようにパーサ定義は通常の Java クラス定義と同様であり、フィールド や メソッド の定義を行うこともできます。
ここではパーサのパッケージ名、クラス名のみ定義しています。

#### 2. 字句規則

生成するパーサの使用する字句規則は `.jj` ファイル内で以下のように定義します。

- (1) 字句 (token) は `TOKEN` 以下に `<token_name: "正規表現">` の形式で指定する
- (2) `SKIP` で指定したものは token としては生成されず無視される

```
// (1)
// Newline
TOKEN: {
    < NEWLINE: "\n" >
}

// Number
TOKEN: {
    < NUMBER: (["0"-"9"])+ >
}

// (2)
// Whitespaces
SKIP: {
    " "
    | "\t"
    | "\r"
    | "\f"
}
```

#### 3. 構文規則

JavaCC における構文規則定義は初見だと少し見辛く感じます。
ですが慣れてくると `.jj` ファイルのみでかなり柔軟な処理を定義できることがわかります。

BNF で `number ::= <NUMBER> <NEWLINE>` (NUMBER, NEWLINE は上で定義した token) と表される生成規則を考えます。この生成規則は `.jj` ファイル内では以下のように定義されます。

- (1) 各構文規則は Java のメソッドに似た形式で定義される
- (2) `<token_name>` で字句規則で定義した token を指定できる

```
// (1)
void number(): {
} {
    // (2)
    <NUMBER> <NEWLINE>
}
```

ほぼ BNF 通りに構文規則が定義されていることがわかります。

これでただ number を受理するだけのパーサであれば生成できるのですが、さらに JavaCC では '構文主導翻訳スキーム' と (dragon book で) 呼ばれる形式で生成規則を定義することもできます。

例えば number をパースしたあと、さらにその数字を出力したいという場合、上の生成規則をもとに以下のように書けます。

- (1) 最初の中括弧内には生成規則のパースを開始する前に実行したい処理を書く
  - 基本的には変数の宣言を行うことが多い
- (2) パースした token は (1) で宣言した変数へ代入できる
- (3) 2 番目の中括弧内に更に中括弧を加えると、任意のタイミングで指定の処理が行える
  - ここでは number のパースが終わったあとに、 `<NUMBER>` でパースした数字を出力している

```
void number(): {
    // (1)
    Token number;
} {
    // (2)
    number = <NUMBER> <NEWLINE> {
        // (3)
        System.out.println(number.image);
    }
}
```

このように `.jj` ファイルのみでパースだけではなく簡単な処理も行うパーサが生成できます。

#### 4. オプション

`.jj` ファイル内 `options` で JavaCC の挙動に関して設定が行えます。
ここでは詳細を省きますが、[こちら][8]が参考になると思います。

```
options {
    STATIC = false;
    JAVA_UNICODE_ESCAPE = true;
    UNICODE_INPUT = true;
}
```

#### jjtree を利用した解析木の作成

上で構文主導スキームのような形式で任意の処理が行えると書きましたが、とはいえあまり複雑なものを `.jj` ファイルに書くと可読性が下がってしまいます。

一般に JavaCC で解析したい文法は木構造を持つはずなので、この木構造に基づいて処理が行えると嬉しいと思います。
そういった需要に答え jjtree では自動的に解析木を作成し、Visitor パターンで処理を行うための機能が用意されています。

これまでは `src/main/javacc` 以下に `.jj` ファイルを配置していましたが、jjtree を利用する場合 `src/main/jjtree` 以下に `.jjt` ファイルが必要になります。

jjtree の利用例として四則演算式を解析するパーサを考えます。
`.jjt` ファイルの書き方はほぼ `.jj` ファイルと同じですが、一部追加しなければいけない処理が存在します。

- (1) 開始規則に対応するメソッドは `SimpleNode` を返り値とする
- (2) `jjtThis` で 処理中の Node (SimpleNode はこれの実装クラス) を扱えるため、これを返す

```
PARSER_BEGIN (Parser)

package com.tiqwab.example;

public class Parser {
}
PARSER_END (Parser)

// --- Definition of tokens ---

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

// --- Translation Scheme ---

// E -> T ("+" T | "-" T)*
// T -> F ("*" T | "/" F)*
// F -> N | "(" E ")"

// (1)
SimpleNode start(): {
} {
    expression() <NEWLINE> {
        // (2)
        return jjtThis;
    }
}

void expression(): {
} {
    term() ( <ADD> term() | <SUBTRACT> term() )*
}

void term(): {
} {
    factor() ( <MULT> factor() | <DIV> factor() )*
}

void factor(): {
} {
    <NUMBER>
    | <LPAREN> expression() <RPAREN>
}
```

`./gradlew clean build` すると `.jj` ファイルや解析木の節に対応する `Node` インタフェース、その実装クラス `SimpleNode` といったものが自動的に生成されているはずです。

```java
public interface Node {
  public void jjtOpen();
  public void jjtClose();
  public void jjtSetParent(Node n);
  public Node jjtGetParent();
  public void jjtAddChild(Node n, int i);
  public Node jjtGetChild(int i);
  public int jjtGetNumChildren();
  public int getId();
}
```

```java
public class SimpleNode implements Node {
    ...
}
```

生成された解析木の概形は `SimpleNode#dump` で確認することができます。

```java
public class ParserMain {
    public static void main(String[] args) throws Exception {
        Parser parser = new Parser(System.in);
        SimpleNode node = parser.start();
        node.dump("");
    }
}
```

```
> 5 + 2 * 4
start
 expression
  term
   factor
  term
   factor
   factor
```

この解析木に対する Visitor を生成するには `.jjt` ファイル内 以下の option を有効にします。

- (1) 解析木に対する Visitor インタフェースの生成を有効にする
- (2) 各生成規則に対応した `Node` 実装クラスの生成を有効にする

```
options {
    // (1)
    VISITOR = true;
    // (2)
    MULTI = true;
}
```

こうすることで以下のような Visitor が生成されるため、あとは好きな処理をこの実装クラスとして定義できます。

```java
public interface ParserVisitor {
  public Object visit(SimpleNode node, Object data);
  public Object visit(ASTStart node, Object data);
  public Object visit(ASTExpression node, Object data);
  public Object visit(ASTTerm node, Object data);
  public Object visit(ASTFactor node, Object data);
}
```

```java
public class ParserMain {
    public static void main(String[] args) throws Exception {
        Parser parser = new Parser(System.in);
        SimpleNode node = parser.Start();
        node.jjtAccept(new SampleVisitor(), null);
    }
}
```

#### 参考

以下のリンクではいずれも JavaCC の概要、使用方法についてわかりやすく解説されています。

- [JavaCC の使い方][3]
- [Mastering Visitor Pattern by jjtree (JavaCC)][4]
- [JavaCC、構文解析ツリー、および XQuery の文法][5]

<div id="asm" />

### 2. ASM

[ASM][9] は Java bytecode を扱うためのライブラリです。

JVM 上でプログラムを実行するには原始プログラムを `.class` ファイル (実態は JVM の仕様に基づいた Java bytecode と呼ばれるバイナリ) に変換する必要がありますが、これを自分で一から行おうとするとタフな仕事になると思います。

ASM はこうした煩雑な部分をケアし、直感的に Java bytecode を生成するための API を提供しています。

#### javap コマンド

実際に ASM を使用するまえにまずは目的物である `.class` ファイルの内容についての確認方法を押さえておきます。
`.class` ファイルはバイナリであり直接中身を確認するのは辛いので、ここでは javap コマンドを利用することにします。

よくある例ですが以下の Hello World をコンパイルしたのちに、javap で中身を確認してみます。

```java
public class Hello {
    public static void main(String[] args) throws Exception {
        System.out.println("Hello world");
    }
}
```

```
> javap -c -v Hello.class
Classfile /home/nm/workspace/jvm/Hello.class
  Last modified Apr 29, 2017; size 463 bytes
  MD5 checksum 9b3a037746a58e01abc24d960b38030b
  Compiled from "Hello.java"
public class Hello
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#17         // java/lang/Object."<init>":()V
   #2 = Fieldref           #18.#19        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #20            // Hello world
   #4 = Methodref          #21.#22        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #23            // Hello
   #6 = Class              #24            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               Exceptions
  #14 = Class              #25            // java/lang/Exception
  #15 = Utf8               SourceFile
  #16 = Utf8               Hello.java
  #17 = NameAndType        #7:#8          // "<init>":()V
  #18 = Class              #26            // java/lang/System
  #19 = NameAndType        #27:#28        // out:Ljava/io/PrintStream;
  #20 = Utf8               Hello world
  #21 = Class              #29            // java/io/PrintStream
  #22 = NameAndType        #30:#31        // println:(Ljava/lang/String;)V
  #23 = Utf8               Hello
  #24 = Utf8               java/lang/Object
  #25 = Utf8               java/lang/Exception
  #26 = Utf8               java/lang/System
  #27 = Utf8               out
  #28 = Utf8               Ljava/io/PrintStream;
  #29 = Utf8               java/io/PrintStream
  #30 = Utf8               println
  #31 = Utf8               (Ljava/lang/String;)V
{
  public Hello();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void main(java.lang.String[]) throws java.lang.Exception;
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 3: 0
        line 4: 8
    Exceptions:
      throws java.lang.Exception
}
SourceFile: "Hello.java"
```

ここでは `.class` ファイルの読み方の詳細は省略しますが、何となく定数宣言している部分 (Constant pool) やクラス、メソッドを定義している部分で構成されていることがわかるかと思います。

#### ASM で Hello World

上記のように `.class` ファイルはいくつかの要素から構成されています。
ASM を使用する場合、 Constant pool 部分に関しては ASM 側でケアしてもらえるため、ライブラリの利用者はクラスとメソッド定義のみに集中することができます。

ASM による Java bytecode 生成において、中心となるクラスは `ClassWriter` です。
`ClassWriter` を使用したクラス定義の流れは以下のようになると説明されています ([asm4-guide][11])。

```
visit visitSource? visitOuterClass? (visitAnnotation | visitAttribute)\*
( visitInnerClass | visitField | visitMethod)\*
visitEnd
```

同様に メソッド定義の中心となる `MethodVisitor` を使用した流れは以下のようになります。

```
visitAnnotationDefault?
( visitAnnotation | visitParameterAnnotation | visitAttribute)\*
( visitCode
  ( visitTryCatchBlock | visitLabel | visitFrame | visitXxxInsn |
    visitLocalVariable | visitLineNumber)\*
  visitMaxs )?
visitEnd
```

これを踏まえ、上の HelloWorld と同等なものを ASM を使用して書くと以下のようになります。

- (1) `org.objectweb.asm.ClassWriter` でクラスを定義する
- (2) クラスの修飾子や名前、継承元などを定義する
- (3) 'org.objectweb.asm.MethodVisitor` でメソッドを定義する
- (4) 全ての定義終了後、Java bytecode を表す `byte[]` を生成する 
- (5) (4) で生成したバイナリを class ファイルとして吐き出す

```java
package com.tiqwab.example.asm;

import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.util.TraceClassVisitor;

import java.io.File;
import java.io.FileOutputStream;
import java.io.OutputStream;
import java.io.PrintWriter;
import java.nio.file.Files;
import java.nio.file.Paths;

public class HelloWorld {

    /* カレントディレクトリに 'hello ASM' を出力する HelloAsm.class を作成する */
    public static void main(String[] args) throws Exception {
        final String classFileName = "HelloAsm.class";
        final String jbcTextFileName = "HelloAsm.txt";
        final String className = "HelloAsm";

        final int flag = ClassWriter.COMPUTE_MAXS | ClassWriter.COMPUTE_FRAMES;
        // (1)
        ClassWriter cw = new ClassWriter(flag);
        // (2)
        cw.visit(
                Opcodes.V1_8,
                Opcodes.ACC_PUBLIC | Opcodes.ACC_SUPER,
                className,
                null,
                "java/lang/Object",
                null
                );

        // For constructor
        visitConstructor(cw);
        // For main
        visitMain(cw);

        // (4)
        byte[] bin = cw.toByteArray();
        // Output as text
        try (OutputStream os = new FileOutputStream(new File(jbcTextFileName));
             PrintWriter pw = new PrintWriter(os)){
            new ClassReader(bin).accept(new TraceClassVisitor(pw), 0);
        }
        // Output as binary
        // (5)
        Files.write(Paths.get(classFileName), bin);
    }

    /* コンストラクタの生成 */
    private static void visitConstructor(ClassWriter cw) {
        // (3)
        MethodVisitor mv = cw.visitMethod(
                Opcodes.ACC_PUBLIC,
                "<init>",
                "()V",
                null,
                null
        );
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
        mv.visitEnd();
    }

    /* main メソッドの生成 */
    private static void visitMain(ClassWriter cw) {
        MethodVisitor mv = cw.visitMethod(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC,
                "main",
                "([Ljava/lang/String;)V",
                null,
                null
        );
        mv.visitCode();
        mv.visitFieldInsn(
                Opcodes.GETSTATIC,
                "java/lang/System",
                "out",
                "Ljava/io/PrintStream;"
        );
        mv.visitLdcInsn("hello ASM");
        mv.visitMethodInsn(
                Opcodes.INVOKEVIRTUAL,
                "java/io/PrintStream",
                "println",
                "(Ljava/lang/String;)V",
                false
        );
        mv.visitInsn(Opcodes.RETURN);
        mv.visitMaxs(0,0);
        mv.visitEnd();
    }

}
```

細かい違いはありますが、クラスやメソッド定義部分は javap で出力されたものと類似しています。
ですので簡単なものならば javap の出力を確認しつつ、ASM で対応する `ClassWriter`、`MethodVisitor` のメソッドを並べていくというやり方ができるかなと思います。

#### StackMapTable の自動計算

`.class` ファイル内のメソッド定義部分には instruction だけではなく 0 以上の stack map frame で構成される StackMapTable と呼ばれるものを含める必要があります ([4.7.4 in jvm specification][14])。

StackMapTable に関する理解は怪しいのですが、用途としては JVM が処理中に実行する Java bytecode の正当性チェックのパフォーマンスを上げるために、事前に (想定される?) frame 中の operand stack に関する情報の計算を行いメソッド定義の一部としている...という感じのようです。

これを自分で計算するのはかなり手間 (詳細を理解するのが難しい) と思うのですが、有り難いことに ASM ではこの部分を自動で計算し、生成する `.class` ファイルに含めるようにできます。

そのためには以下の操作を行います。

- (1) `ClassWriter` 生成時に flag として `COMPUTE_MAXS, COMPUTE_FRAMES` を渡す
- (2) メソッド定義後、 `MethodVisitor#visitMaxs` を呼ぶ
  - このとき引数は使われないので任意でよい

```java
// (1)
final int flag = ClassWriter.COMPUTE_MAXS | ClassWriter.COMPUTE_FRAMES;
ClassWriter cw = new ClassWriter(flag);
...
MethodVisitor mv = cw.visitMethod(...);
mv.visitCode();
...
// (2)
mv.visitMaxs(0,0);
```

ASM のマニュアルによるとこの処理を行うことで bytecode の生成速度は 2 倍近く落ち得るとのことでしたが、それが許される状況であれば積極的に使用したい機能ではないでしょうか。

#### 参考

JVM の実行系および ASM についての概要は以下を参考にしています。

- [asm4-guide][11]
  - ASM の詳細をまとめた pdf
- [Stack on JavaVM][12]
  - JVM の処理系について
- [Java のクラスファイルを javap とバイナリエディタで読む][13]
  - クラスファイルの概要について

[1]: https://github.com/eclipse/golo-lang/ "golo-lang"
[2]: https://github.com/johnmartel/javaccPlugin "javaccPlugin"
[3]: http://cis.k.hosei.ac.jp/~nakata/lectureCompiler/JavaCC/index.html "JavaCC の使い方"
[4]: http://www002.upp.so-net.ne.jp/ys_oota/jjtree/ "Mastering Visitor Pattern by jjtree (JavaCC)"
[5]: https://www.ibm.com/developerworks/jp/xml/library/x-javacc1/ "JavaCC、構文解析ツリー、および XQuery の文法"
[6]: https://javacc.org/ "JavaCC"
[7]: https://ja.wikipedia.org/wiki/Yacc "Yacc"
[8]: http://cis.k.hosei.ac.jp/~nakata/lectureCompiler/JavaCC/index.html
[9]: http://asm.ow2.org/ "ASM"
[10]: http://dev.classmethod.jp/server-side/java/classfile-reading/
[11]: http://download.forge.objectweb.org/asm/asm4-guide.pdf "ASM 4.0 A Java bytecode engineering library"
[12]: https://www.slideshare.net/skrb/stack-on-javavm-4932723 "Stack on JavaVM"
[13]: http://dev.classmethod.jp/server-side/java/classfile-reading/
[14]: https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.7.4
[15]: https://github.com/tiqwab/simple-jvm-lang
