---
layout: post
title: Hotspot JVM の interpreter と tos について
tags: "jvm, hotspot jvm, openjdk"
comments: true
---

元にした OpenJDK ソース: [jdk11u][1] の changeset 51892:e86c5c20e188 です。

### OpenJDK の interpreter とは

- Java byte code の interpreter
- (Java) メソッドはまず interpreter で実行される
- (条件を満たすと) JIT コンパイルで生成されたネイティブコードを実行する

このあたりについては [Hotspot VMの基本構造を理解する][3] の図9 がわかりやすくまとまっています。

簡単な Java byte code 例として int 同士の加算を行うメソッドを取り上げます。

```java
class Example1 {
    public static int add() {
        int x = 1;
        int y = 2;
        return x + y;
    }
}
```

これをコンパイルし class ファイルから add メソッドに相当する部分を抜き出したものが以下になります。
JVM はスタックマシンなので各 Java byte code では入力や出力をスタックを介してやり取りしています。

```
$ javac Example1.java
$ javap -c -v Example1.class
  ...
  public static int add();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=0
         0: iconst_1 // (int) 1 をスタックに push
         1: istore_0 // スタックから pop してローカル変数 0 番へ
         2: iconst_2 // (int) 2 を push
         3: istore_1 // pop してローカル変数 1 番へ
         4: iload_0  // ローカル変数 0 番 を push
         5: iload_1  // ローカル変数 1 番を push
         6: iadd     // スタックから val2, val1 と pop して (val1 + val2) を push
         7: ireturn  // スタック先頭の値を返す
      LineNumberTable:
        line 10: 0
        line 11: 2
        line 12: 4
  ...
```

### 2 種類の Java byte code interpreter

OpenJDK の [RuntimeOverview#Interpreter][2] に記載されているように、2 種類の実装が存在します。

- Cpp Interpreter
- Template Interpreter

### Cpp Interpreter

CppInterpreter とは OpenJDK 内の interpreter 実装の 1 つです。上の overview で言及している

> a classic switch-statement loop

を採用した実装といえます。

ソースとしては `src/hotspot/share/interpreter/bytecodeInterpreter.cpp` にメインのループ部が存在します。

```c++
// CppInterpreter 内で BytecodeInterpreter::run を使用
//
// 一部省略 & 単純化
void
BytecodeInterpreter::run(interpreterState istate) {
    ...
    register address pc = istate->bcp(); // bcp = byte code pointer
    ...
    while (1) {
        opcode = *pc;

        switch (opcode)
        {
        ...
        CASE(_i2f):       /* convert top of stack int to float */
            SET_STACK_FLOAT(VMint2Float(STACK_INT(-1)), -1);
            UPDATE_PC_AND_CONTINUE(1);
        ...
        CASE(_ireturn):
            {
            // Allow a safepoint before returning to frame manager.
            SAFEPOINT;

            goto handle_return;
            }
        }
        ...
    }
    ...
    handle_return: {
        ...
    }
}
```

Cpp Interpreter は割と直感的ですが性能があまり良くないために開発用ビルドでフラグを有効にしない限り使用されません。通常は interpreter の実装として後述の Template Interpreter が使用されます。

### Template Interpreter

- 各 Java byte code について専用のネイティブコードを (JVM 起動直後に) 生成する
- 生成したネイティブコードの間を飛び回る感じになる

生成されるネイティブコードを見るには開発用に用意されている、`-XX:+PrintInterpreter` フラグ付きで JVM を起動します (ビルドの方法は [以前の記事][4] 参照)。

```
$ java -XX:+PrintInterpreter Hello | tee interpreter.log
```

interpreter.log には interpreter 用に生成されたアセンブリが出力されます。

例えば arraylength という Java byte code 用に生成されるネイティブコードを下に抜き出しました (for x86\_64)。
arraylength は 「オペランドスタックから配列オブジェクトへの参照を pop し、その長さを push」する命令です。

```
arraylength  190 arraylength  [0x00007f6d3903c8a0, 0x00007f6d3903c8c0]  32 bytes

  # (必要なら) operand の取得
  0x00007f6d3903c8a0: pop    %rax

  # メインの処理
  0x00007f6d3903c8a1: mov    0xc(%rax),%eax       # 0xc(%rax) に配列オブジェクトならば長さが格納されている

  # 次の Java byte code を読む準備
  0x00007f6d3903c8a4: movzbl 0x1(%r13),%ebx       # ebx レジスタに次の Java byte code を保存
  0x00007f6d3903c8a9: inc    %r13                 # bcp (byte code pointer) の increment
  0x00007f6d3903c8ac: movabs $0x7f6d4ed7bd60,%r10 # r10 レジスタに dispatch table のベースアドレスを保存
  0x00007f6d3903c8b6: jmpq   *(%r10,%rbx,8)       # 次の Java byte code 用のコードへ jmp
  0x00007f6d3903c8ba: nop
  0x00007f6d3903c8bb: nop
  0x00007f6d3903c8bc: nop
  0x00007f6d3903c8bd: nop
  0x00007f6d3903c8be: nop
  0x00007f6d3903c8bf: nop
```

まずは pop 命令により配列オブジェクトへの参照を rax レジスタにセットしています。これは OpenJDK の場合ヒープ上の配列オブジェクトの先頭アドレスです。コメントで必要なら、としているのは後述の tos を考慮した上でこの命令を行わない場合があるため (次の mov 命令から始める場合があるため) です。

次は `mov 0xc(%rax),%eax` です。コメントで書いた通り arraylength 固有の処理はここだけです。rax レジスタにセットした配列オブジェクトへの参照から長さの情報を引っ張ってきて eax レジスタにセットしています。配列オブジェクトの場合 (実行環境や起動時に JVM に渡すオプションで変わりますが) 先頭 12 byte 目から 4 bytes に長さの情報が入ることになっています。ここで取得した情報をスタックに push するのではなく eax レジスタにセットする、という点も後述の tos についての箇所で説明します。

mov 命令のあとは次の Java byte code を処理するための準備で、

- bcp の increment
  - bcp は Java byte code 用のプログラムカウンタのようなもの
- 次の Java byte code 用のコードへの jmp
  - 各 Java byte code に振られている番号 (+α) の情報で分岐無しに飛べる

といったことをしています。
このあたりの処理は arraylength に限ったものではなく各 Java byte code でも行うお決まりのパターンです。

Template Interpreter による処理の様子を図にするとこんな感じです。
例えば aload\_1, arraylength, istore\_2 という Java byte code を処理するにはまず alad\_1 の native code を実行し、arraylength 用の native code へ jmp, それを実行したら istore\_2 用の native code へ jmp ... となります。

<figure>
  <img
    src="/images/openjdk-interpreter/fig1.jpg"
    title="Abstraction of Template Interpreter"
    alt="Abstraction of Template Interpreter"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
  <figcaption>Template Interpreter による処理の様子</figcaption>
</figure>

ここまでの内容は OpenJDK のソースでは

- Template
- TemplateTable
- DispatchTable

あたりが該当する箇所になります。

#### Template

in `src/hotspot/share/interpreter/templateTable.hpp`

- interpreter 用 native code の generator

```cpp
 class Template {
   ...
   
   typedef void (*generator)(int arg);
 
   int       _flags;        // describes interpreter template properties (bcp unknown)
   TosState  _tos_in;      // tos cache state before template execution
   TosState  _tos_out;     // tos cache state after  template execution
   generator _gen;         // template code generator
   int       _arg;         // argument for template code generator
   
   ...
 }
```

#### TemplateTable

in `src/hotspot/share/interpreter/templateTable.hpp`

- Template をまとめたもの
- 各 Java byte code についてどのような native code を生成するかは TemplateTable の initialize メソッドから追える
  - JVM 起動時にこのメソッドが呼ばれる

```cpp
 void TemplateTable::initialize() {
   ...
   
   //                                 interpr. templates
   // Java spec bytecodes             ubcp|disp|clvm|iswd  in    out   generator             argument
   
     def(Bytecodes::_arraylength    , ____|____|____|____, atos, itos, arraylength         ,  _           );
     
   ...
 }
```

各 generator は CPU 依存なので `src/hotspot/cpu/<arch>/templateTable_<arch>.cpp` のような命名のソースに存在します。例えば Intel x86 の場合 `src/hotspot/cpu/x86/templateTable_x86.cpp` です。

`__` の実態は InterpreterMacroAssembler と呼ばれるものでネイティブコードを生成するために使用されています。

```cpp
void TemplateTable::arraylength() {
  transition(atos, itos);
  __ null_check(rax, arrayOopDesc::length_offset_in_bytes());
  __ movl(rax, Address(rax, arrayOopDesc::length_offset_in_bytes()));
}
```

この中には上のアセンブリでいう「メインの処理」しかありません (`__movl` がそれ)。
(transition はただの assetion で、null\_check はデフォルト何もしないっぽい)

#### DispatchTable

in `src/hotspot/share/interpreter/templateInterpreter.hpp`

- 各 Java byte code interpreter 用に生成したコードのエントリポイント (のアドレス) を管理
  - 256 (< Java byte code の種類) x tos (Top Of Stack) 10 のアドレスを `_table` に持つ
- `set_entry` でエントリポイントの設定
- `table_for` で tos 毎のテーブルのベースアドレスを取得

```cpp
 class DispatchTable {
  public:
   enum { length = 1 << BitsPerByte }; // an entry point for each byte value
  public:
   address _table[number_of_states][length]; // dispatch tables, indexed by tosca and bytecode

   ...

   void       set_entry(int i, EntryPoint& entry);     // set entry point for a given bytecode i
   address*   table_for(TosState state)          { return _table[state]; }
 }
```

このテーブルの初期化は`TemplateInterpreterGenerator::set_entry_points_for_all_bytes` で行われています。

### tos (Top Of Stack) について

- 各 Java byte code 処理前後でスタックの先頭値の型として期待するもの
  - JVM はスタックマシンなので各命令の入力、出力の型ともいえる
  - ここでは入力側の tos を in tos, 出力側を out tos と呼ぶ
- Java byte code が決まれば in tos, out tos も決まる
  - 例えば arraylength なら in tos = atos, out tos = itos
- tos は全部で 10 種類
  - btos (byte, bool)
  - ztos (byte, bool)
  - ctos (char)
  - stos (short)
  - itos (int)
  - ltos (long)
  - ftos (float)
  - dtos (double)
  - atos (object)
  - vtos (void)

tos は JVM spec には存在しない概念であり、OpenJDK 独自の話です。
OpenJDK では tos を考えることで interpreter の性能向上を行っています。

JVM がスタックマシンの挙動をすることから単純に考えるとネイティブスタックを使用して各 Java byte code の入力、出力の受け渡しを行いたくなりますが、もしこの部分をレジスタで行えればメモリにアクセスしにいかずに済む分 interpreter の性能向上に繋がります。とはいえ常にレジスタでやり取りができるとは限らず、その条件の判定のために連続する Java byte code 間の tos を比較します。

考え方としては

- とりあえず計算値をレジスタに置いて次の Java byte code 処理へ jmp する
  - このとき tos の判定により、jmp 先が変わる
- 次の Java byte code を処理するとき
  - うまくレジスタでやり取りができていればそのままそれを使用する
  - 駄目だったら渡された値をスタックに置き直す or スタックから値を取得する

という感じです。

例えば aload\_1, arraylength という組み合わせの場合、

- aload\_1 の in tos は vtos, out tos は atos
- arraylength の in tos は atos, out tos は itos

であり、前の Java byte code の out tos と後の Java byte code の in tos が一致するのでレジスタで受け渡しをして構わないと判断されます。

一方で (極端な例ですが) aload\_1, nop, arraylength という組み合わせだった場合、 

- nop の in tos は vtos, out tos も vtos

であり、この場合 aload\_1 の out tos と nop の in tos は一致しないため、レジスタでの受け渡しに失敗しています。そのため nop ではまず後続のためにレジスタ上の値をスタックに置き直してから自身の処理を行います (nop なので何もしませんが)。nop, arraylength 間もやはり tos が一致せず、arraylength では冒頭でスタックから値を取得した上で自身の処理を行います。

<figure>
  <img
    src="/images/openjdk-interpreter/fig2.jpg"
    title="Optimization by comparing tos"
    alt="Optimization by comparing tos"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
  <figcaption>tos の比較で値の受け渡し方法を変えている</figcaption>
</figure>

この部分の native code を生成するのは以下の処理です。

```cpp
void TemplateInterpreterGenerator::set_short_entry_points(Template* t, address& bep, address& cep, address& sep, address& aep, address& iep, address& lep, address& fep, address& dep, address& vep) {
  assert(t->is_valid(), "template must exist");
  switch (t->tos_in()) {
    case btos:
    case ztos:
    case ctos:
    case stos:
      ShouldNotReachHere();  // btos/ctos/stos should use itos.
      break;
    case atos: vep = __ pc(); __ pop(atos); aep = __ pc(); generate_and_dispatch(t); break;
    case itos: vep = __ pc(); __ pop(itos); iep = __ pc(); generate_and_dispatch(t); break;
    case ltos: vep = __ pc(); __ pop(ltos); lep = __ pc(); generate_and_dispatch(t); break;
    case ftos: vep = __ pc(); __ pop(ftos); fep = __ pc(); generate_and_dispatch(t); break;
    case dtos: vep = __ pc(); __ pop(dtos); dep = __ pc(); generate_and_dispatch(t); break;
    case vtos: set_vtos_entry_points(t, bep, cep, sep, aep, iep, lep, fep, dep, vep);     break;
    default  : ShouldNotReachHere();                                                 break;
  }
}
```

これを見ると結局は以下の 4 パターンに一般化できます。

- Xtos to Xtos (Xtos は vtos 以外のいずれか)
  - レジスタで受け渡された値を使って処理を進める。最も効率的
- vtos to Xtos
  - レジスタでの受け渡しに失敗 (jmp 前のテンプレートでは tos を用意しないので)
  - スタックに値があるはずなのでそれを pop してから処理を開始する。 
- Xtos to vtos
  - レジスタでの受け渡しに失敗 (jmp 先のテンプレートでは tos を使用しないので)
  - 後続の Java byte code の処理で使用する可能性があるので、レジスタで渡された値をスタックに入れ直してから処理を進める
- vtos to vtos
  - レジスタでの受け渡しをしない
  - 実際のソースではパターンとしては Xtos to vtos と同じとみなしている (スタックに入れ直さないだけ)

最後に jmp 先のアドレスについて DispatchTable でどのように管理されているかを arraylength を例に確認します。
以下は gdb から確認した情報ですが、具体的なアドレス値は実行毎に変わり得ます。

(この情報を gdb で見るなら `TemplateInterpreter::_normal_table._table[8][190]` とかで)

- `_table[atos][190]` には 0x00007f6d3903c8a1
  - 例えば aload\_1, arraylength という並びでの aload\_1 からの jmp 先
  - 直前の aload\_1 の処理で配列オブジェクトの参照値をレジスタに置いているはずなのでそのまま使う
- `_table[vtos][190]` には 0x00007f6d3903c8a0
  - 例えば aload\_1, nop, arraylength という並びでの nop からの jmp 先
  - 直前の nop では何も出力していないが、それより前の aload\_1 の参照値をスタックに積んだはずなのでそれを pop するところから始める
- それ以外はエラー処理を行うルーチンへ飛ぶ (はず)

```
arraylength  190 arraylength  [0x00007f6d3903c8a0, 0x00007f6d3903c8c0]  32 bytes
  0x00007f6d3903c8a0: pop    %rax
  0x00007f6d3903c8a1: mov    0xc(%rax),%eax       # 0xc(%rax) に配列オブジェクトならば長さが格納されている
  0x00007f6d3903c8a4: movzbl 0x1(%r13),%ebx       # ebx レジスタに次の Java byte code を保存
  0x00007f6d3903c8a9: inc    %r13                 # bcp (byte code pointer) の increment
  0x00007f6d3903c8ac: movabs $0x7f6d4ed7bd60,%r10 # r10 レジスタに dispatch table のベースアドレスを保存
  0x00007f6d3903c8b6: jmpq   *(%r10,%rbx,8)       # 次の Java byte code 用のコードへ jmp
  0x00007f6d3903c8ba: nop
  0x00007f6d3903c8bb: nop
  0x00007f6d3903c8bc: nop
  0x00007f6d3903c8bd: nop
  0x00007f6d3903c8be: nop
  0x00007f6d3903c8bf: nop
```

aload\_1 側のアセンブリは

```
aload_1  43 aload_1  [0x00007fffe101f520, 0x00007fffe101f580]  96 bytes

  0x00007fffe101f520: push   %rax
  0x00007fffe101f521: jmpq   0x00007fffe101f55f
  0x00007fffe101f526: sub    $0x8,%rsp
  0x00007fffe101f52a: vmovss %xmm0,(%rsp)
  0x00007fffe101f52f: jmpq   0x00007fffe101f55f
  0x00007fffe101f534: sub    $0x10,%rsp
  0x00007fffe101f538: vmovsd %xmm0,(%rsp)
  0x00007fffe101f53d: jmpq   0x00007fffe101f55f
  0x00007fffe101f542: sub    $0x10,%rsp
  0x00007fffe101f546: mov    %rax,(%rsp)
  0x00007fffe101f54a: movabs $0x0,%r10
  0x00007fffe101f554: mov    %r10,0x8(%rsp)
  0x00007fffe101f559: jmpq   0x00007fffe101f55f
  0x00007fffe101f55e: push   %rax
  0x00007fffe101f55f: mov    -0x8(%r14),%rax
  0x00007fffe101f563: movzbl 0x1(%r13),%ebx
  0x00007fffe101f568: inc    %r13
  0x00007fffe101f56b: movabs $0x7ffff7b86d60,%r10 # r10 レジスタに dispatch table のベースアドレスを保存
  0x00007fffe101f575: jmpq   *(%r10,%rbx,8)
  0x00007fffe101f579: nop
  0x00007fffe101f57a: nop
  0x00007fffe101f57b: nop
  0x00007fffe101f57c: nop
  0x00007fffe101f57d: nop
  0x00007fffe101f57e: nop
  0x00007fffe101f57f: nop
```

という感じですが、jmp 先の計算に atos 用の `table[atos]` を使用しています。 `jmp *(%r10,%rbx,8)` で `table[atos][190]` の指すアドレスへ jmp しています。このように DispatchTable では tos, bytecode 毎に jmp 先を用意しているので実行時に一切分岐なしで処理することができています。

nop の場合も同様でこちらは `table[vtos]` を使用しており、jmp 先は `table[vtos][190]` になります。

```
nop  0 nop  [0x00007fffe101c6a0, 0x00007fffe101c700]  96 bytes

  0x00007fffe101c6a0: push   %rax
  0x00007fffe101c6a1: jmpq   0x00007fffe101c6df
  0x00007fffe101c6a6: sub    $0x8,%rsp
  0x00007fffe101c6aa: vmovss %xmm0,(%rsp)
  0x00007fffe101c6af: jmpq   0x00007fffe101c6df
  0x00007fffe101c6b4: sub    $0x10,%rsp
  0x00007fffe101c6b8: vmovsd %xmm0,(%rsp)
  0x00007fffe101c6bd: jmpq   0x00007fffe101c6df
  0x00007fffe101c6c2: sub    $0x10,%rsp
  0x00007fffe101c6c6: mov    %rax,(%rsp)
  0x00007fffe101c6ca: movabs $0x0,%r10
  0x00007fffe101c6d4: mov    %r10,0x8(%rsp)
  0x00007fffe101c6d9: jmpq   0x00007fffe101c6df
  0x00007fffe101c6de: push   %rax
  0x00007fffe101c6df: movzbl 0x1(%r13),%ebx
  0x00007fffe101c6e4: inc    %r13
  0x00007fffe101c6e7: movabs $0x7ffff7b87560,%r10 # r10 レジスタに dispatch table のベースアドレスを保存
  0x00007fffe101c6f1: jmpq   *(%r10,%rbx,8)
  0x00007fffe101c6f5: nop
  0x00007fffe101c6f6: nop
  0x00007fffe101c6f7: nop
  0x00007fffe101c6f8: int3   
  0x00007fffe101c6f9: int3   
  0x00007fffe101c6fa: int3   
  0x00007fffe101c6fb: int3   
  0x00007fffe101c6fc: int3   
  0x00007fffe101c6fd: int3   
  0x00007fffe101c6fe: int3   
  0x00007fffe101c6ff: int3   
```

[1]: http://hg.openjdk.java.net/jdk-updates/jdk11u
[2]: https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html#Interpreter|outline
[3]: https://www.atmarkit.co.jp/ait/articles/0403/11/news096_2.html
[4]: {% post_url 2019-07-08-openjdk-build %}
