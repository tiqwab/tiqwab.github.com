---
layout: post
title: Hotspot JVM の invokestatic と return 命令を読む
tags: "jvm, hotspot jvm, openjdk"
comments: true
---

前回 [Hotspot JVM の frame 構造][1] について確認したのですが、実際にそれを使用するメソッドの呼出やそこから戻ってくる部分の処理が見れなかったので、invokestatic と return を題材にそのあたりを確認します。

---

### invokestatic

invokestatic 用の native code を作成しているのは以下のメソッドです。

```c++
// in src/hotspot/cpu/x86/templateTable_x86.cpp

void TemplateTable::invokestatic(int byte_no) {
  prepare_invoke(byte_no, rbx);  // get f1 Method*
  // do the call
  __ profile_call(rax);
  __ profile_arguments_type(rax, rbx, rbcp, false);
  __ jump_from_interpreted(rbx, rax);
}
```

profile (method data pointer) まわりを無視すると

- prepare\_invoke
  - 保存したいレジスタの情報を frame に保存
  - レジスタの用意
  - return address の用意
    - invoke 対象のメソッドの interpret が終わったときの jmp 先になる
    - `-XX:+PrintInterpreter` の invoke return entry points 内のどこか
- jump\_from\_interpreted
  - 現 frame の last sp に rsp をセット
    - last sp は frame 作成時には 0x00 であり、このタイミングでセットされている
  - jmp

といった処理が行われます。

#### prepare\_invoke

全体を追うのは大変なので invokestatic 関係で大事そうなところのみ抜き出しています。

```c++
// in src/hotspot/cpu/x86/templateTable_x86.cpp

void TemplateTable::prepare_invoke(int byte_no,
                                   Register method,  // linked method (or i-klass)
                                   Register index,   // itable index, MethodType, etc.
                                   Register recv,    // if caller wants to see it
                                   Register flags    // if caller wants to test it
                                   ) {
  ...
  if (flags == noreg)  flags = rdx;
  ...
  // save 'interpreter return address'
  __ save_bcp();
  load_invoke_cp_cache_entry(byte_no, method, index, flags, is_invokevirtual, false, is_invokedynamic);
  ...
  // compute return type
  __ shrl(flags, ConstantPoolCacheEntry::tos_state_shift);
  // Make sure we don't need to mask fjlags after the above shift
  ConstantPoolCacheEntry::verify_tos_state_shift();
  // load return address
  {
    const address table_addr = (address) Interpreter::invoke_return_entry_table_for(code);
    ExternalAddress table(table_addr);
    LP64_ONLY(__ lea(rscratch1, table));
    LP64_ONLY(__ movptr(flags, Address(rscratch1, flags, Address::times_ptr)));
  }
  
  // push return address
  __ push(flags);
  ... 
}
```

以下のような処理が行われているようです。

- save\_bcp
  - bcp register (x86\_64 の場合 r13) を frame 内の bcp 領域に退避
  - interpret 中はレジスタしか更新していないのでここで最新の値を反映する
- load\_invoke\_cp\_cache\_entry
  - method register (x86\_64 の場合 rbx) に invoke 対象の `Method*` を設定
- compute return type
  - flags register から tos にあたる情報を抜き出し
- load return address
- push return address
  - この 2 つで invoke したメソッド終了時の戻り先を計算しスタックにプッシュしている
  - 戻り先は専用のテーブルで管理され、invoke 命令の種類 (e.g. invokestatic) や tos 等によって決まる

#### jump\_from\_interpreted

```c++
void InterpreterMacroAssembler::prepare_to_jump_from_interpreted() {
  // set sender sp
  lea(_bcp_register, Address(rsp, wordSize));
  // record last_sp
  movptr(Address(rbp, frame::interpreter_frame_last_sp_offset * wordSize), _bcp_register);
}

// Jump to from_interpreted entry of a call unless single stepping is possible
// in this thread in which case we must call the i2i entry
void InterpreterMacroAssembler::jump_from_interpreted(Register method, Register temp) {
  prepare_to_jump_from_interpreted();
  ...
  jmp(Address(method, Method::from_interpreted_offset()));
}
```

- Method::from\_interpreted\_offset に入っているのは例えば TemplateInterpreterGenerator の generate\_normal\_entry で生成されたエントリポイントのアドレス
  - ここで新たな frame が用意されてからメソッドの interpret をする

実際にここまでの挙動を gdb で確認してみます。
例として以下のような Java プログラムを利用します。

```java
class InvokeStatic {
    public static void f(int a, int b) {
        int sum = a + b;
        System.out.println("sum: " + sum);
    }

    public static void main(String[] args) {
        int x = 1;
        int y = 2;
        f(1 + 2, 2 + 2);
    }
}
```

```
$ javap -c -v InvokeStatic.class
...
  InvokeStatic();
    descriptor: ()V
    flags:
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void f(int, int);
    descriptor: (II)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=3, args_size=2
         0: iload_0
         1: iload_1
         2: iadd
         3: istore_2
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: new           #3                  // class java/lang/StringBuilder
        10: dup
        11: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
        14: ldc           #5                  // String sum:
        16: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        19: iload_2
        20: invokevirtual #7                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        23: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        26: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        29: return
      LineNumberTable:
        line 3: 0
        line 4: 4
        line 5: 29

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: iconst_1
         1: istore_1
         2: iconst_2
         3: istore_2
         4: iconst_3
         5: iconst_4
         6: invokestatic  #10                 // Method f:(II)V
         9: return
      LineNumberTable:
        line 8: 0
        line 9: 2
        line 10: 4
        line 11: 9
...
```

main メソッドから f メソッドを呼び出す invokestatic 実行時 (invokestatic の native code 開始時) のスタックの様子は以下のようになります。main メソッドの frame が確認できています (sp = stack pointer, fp = frame pointer, bcp = byte code pointer)。

```
# breakpoint at the beginning of invokestatic
(gdb) p $rbp
$1 = (void *) 0x7ffff59ed8d8
(gdb) p $rsp
$2 = (void *) 0x7ffff59ed880
(gdb) p ($rbp - $rsp)
$3 = 88
(gdb) x /128xb $rsp
0x7ffff59ed880:	0x04	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # operand stack <- rsp
0x7ffff59ed888:	0x03	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # operand stack
0x7ffff59ed890:	0x90	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # expression stack
0x7ffff59ed898:	0x48	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00 # bcp
0x7ffff59ed8a0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # pointer to locals
0x7ffff59ed8a8:	0xc8	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00 # constant pool cache
0x7ffff59ed8b0:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # methodData
0x7ffff59ed8b8:	0x18	0x54	0x6f	0x19	0x07	0x00	0x00	0x00 # mirror
0x7ffff59ed8c0:	0x60	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00 # Method*
0x7ffff59ed8c8:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # last sp
0x7ffff59ed8d0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # old sp 1
0x7ffff59ed8d8:	0x60	0xd9	0x9e	0xf5	0xff	0x7f	0x00	0x00 # old fp 1 <- rbp
0x7ffff59ed8e0:	0xf3	0x09	0x00	0xe1	0xff	0x7f	0x00	0x00 # return address 1
0x7ffff59ed8e8:	0x02	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # local
0x7ffff59ed8f0:	0x01	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # local
0x7ffff59ed8f8:	0x60	0x59	0x6f	0x19	0x07	0x00	0x00	0x00 # param
```

以下は normal\_entry へ jmp する直前のスタックの様子です。

```
# breakpoint at jmp (to method entrypoint) of invokestatic
(gdb) p $rbp
$4 = (void *) 0x7ffff59ed8d8
(gdb) p $rsp
$5 = (void *) 0x7ffff59ed878
(gdb) x /136xb $rsp
0x7ffff59ed878:	0xa7	0x9f	0x00	0xe1	0xff	0x7f	0x00	0x00 # return address 2 <- rsp
0x7ffff59ed880:	0x04	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # operand stack <- r13
0x7ffff59ed888:	0x03	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # operand stack
0x7ffff59ed890:	0x90	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed898:	0x4e	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8a0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed8a8:	0xc8	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8b0:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8b8:	0x18	0x54	0x6f	0x19	0x07	0x00	0x00	0x00
0x7ffff59ed8c0:	0x60	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8c8:	0x80	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # last sp
0x7ffff59ed8d0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # old sp 1
0x7ffff59ed8d8:	0x60	0xd9	0x9e	0xf5	0xff	0x7f	0x00	0x00 # old fp 1 <- rbp
0x7ffff59ed8e0:	0xf3	0x09	0x00	0xe1	0xff	0x7f	0x00	0x00 # return address 1
0x7ffff59ed8e8:	0x02	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8f0:	0x01	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8f8:	0x60	0x59	0x6f	0x19	0x07	0x00	0x00	0x00
```

- last sp に rsp が設定されている
  - この時点の rsp + word のアドレスを指す
  - 同じ値が r13 レジスタにも入っている
- 計算した return address 2 がスタックに push されている

このあと normal\_entry で新しい frame が作成され、f メソッドの interpret を開始した時点のスタックは以下のようになります。図でメモリアドレス 0x7ffff59ed820 - 0x7ffff59ed878 のあたりが新 frame 部分です。

```
# breakpoint at the beginning of interpretation of method 'f'
(gdb) p $rbp
$6 = (void *) 0x7ffff59ed868
(gdb) p $rsp
$7 = (void *) 0x7ffff59ed820
(gdb) p (((long) 0x7ffff59ed8f8) - ((long) $rsp))
$9 = 216
(gdb) x /216xb $rsp
(以下は f メソッドの frame)
0x7ffff59ed820:	0x20	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # <- rsp
0x7ffff59ed828:	0x80	0x53	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed830:	0x88	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed838:	0xc8	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed840:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed848:	0x18	0x54	0x6f	0x19	0x07	0x00	0x00	0x00
0x7ffff59ed850:	0xa8	0x53	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed858:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed860:	0x80	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # old sp2
0x7ffff59ed868:	0xd8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00 # old fp2 <- rbp
0x7ffff59ed870:	0xa7	0x9f	0x00	0xe1	0xff	0x7f	0x00	0x00 # return address 2
0x7ffff59ed878:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # local
(以下は main メソッドの frame)
0x7ffff59ed880:	0x04	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # param (<- old sp2)
0x7ffff59ed888:	0x03	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # param
0x7ffff59ed890:	0x90	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed898:	0x4e	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8a0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed8a8:	0xc8	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8b0:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8b8:	0x18	0x54	0x6f	0x19	0x07	0x00	0x00	0x00
0x7ffff59ed8c0:	0x60	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8c8:	0x80	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed8d0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed8d8:	0x60	0xd9	0x9e	0xf5	0xff	0x7f	0x00	0x00 # (<- old fp2)
0x7ffff59ed8e0:	0xf3	0x09	0x00	0xe1	0xff	0x7f	0x00	0x00
0x7ffff59ed8e8:	0x02	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8f0:	0x01	0x00	0x00	0x00	0x00	0x00	0x00	0x00
```

### return

return 用の native code を作成しているのは以下のメソッドです。
どうやら bytecode が `_return_register_finalizer` である場合は追加の処理がありますがここでは無視しています。

```c++
// in src/hotspot/cpu/x86/templateTable_x86.cpp

void TemplateTable::_return(TosState state) {
  ...
  // Narrow result if state is itos but result type is smaller.
  // Need to narrow in the return bytecode rather than in generate_return_entry
  // since compiled code callers expect the result to already be narrowed.
  if (state == itos) {
    __ narrow(rax);
  }
  __ remove_activation(state, rbcp);
  __ jmp(rbcp);
}
```

- メソッド f 用の frame を解放する
- rbp を メソッド main の interpret 時のものに戻す (old fp2)
- rsp を old sp2 に戻す

といったことが行われます。

そのあと、rbcp (r13) レジスタに invokestatic でセットした return address 2 がセットされるのでそこへ jmp します。

```
# breakpoint at the return address 2 (after jmp)
(gdb) p $rbp
$203 = (void *) 0x7ffff59ed8d8
(gdb) p $rsp
$204 = (void *) 0x7ffff59ed880
(gdb) x /128xb $rsp
(main メソッドの frame)
0x7ffff59ed880:	0x04	0x00	0x00	0x00	0x00	0x00	0x00	0x00 # <- rsp
0x7ffff59ed888:	0x03	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed890:	0x90	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed898:	0x4e	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8a0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed8a8:	0xc8	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8b0:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8b8:	0x18	0x54	0x6f	0x19	0x07	0x00	0x00	0x00
0x7ffff59ed8c0:	0x60	0x54	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ed8c8:	0x80	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed8d0:	0xf8	0xd8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ed8d8:	0x60	0xd9	0x9e	0xf5	0xff	0x7f	0x00	0x00 # <- rbp
0x7ffff59ed8e0:	0xf3	0x09	0x00	0xe1	0xff	0x7f	0x00	0x00
0x7ffff59ed8e8:	0x02	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8f0:	0x01	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff59ed8f8:	0x60	0x59	0x6f	0x19	0x07	0x00	0x00	0x00
```

jmp 後の return entry points 内の処理では frame からレジスタに値を戻したり (e.g. bcp\_registers, local\_registers) してから dispath\_next で main 内 invokestatic の次の Java byte code から interpret を再開します。

---

まとめると

```
invokestatic (Java byte code)
↓
normal entry
↓
Java byte codes in the invoked method
↓
return (Java byte code)
↓
return entry points
↓
continue to interpret of the caller (who executed invokestatic) method
```

という感じでした。


[1]: {% post_url 2019-08-25-openjdk-frame %}