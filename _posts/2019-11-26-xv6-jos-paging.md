---
layout: post
title: xv6 と JOS のページング
tags: "jos,xv6"
comments: true
---

[前回][3] から引き続き xv6 と JOS を触っています。
ここではそれぞれの仮想アドレス空間と fork 時のページテーブル作成についてまとめています。
大まかには似ているのですが、違うところもあるので太字で強調しています。

### xv6 のアドレス空間

<figure>
  <img
    src="/images/xv6-jos-page-table/figure1.jpg"
    title="xv6 page mapping"
    alt="xv6 page mapping"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

(画像は [xv6 book の Figure 2-2][2] からの引用)

- KERNBASE (0x80000000) より上をカーネル用の領域にしている
  - どのプロセスもカーネル領域のマッピング (図中青色の領域) を共通に持つ
  - カーネル領域では仮想アドレスから KERNBASE を引くことで物理アドレスを得られる
    - マクロで表すと V2P が仮想アドレスから物理アドレスへの変換、P2V が逆

```c
#define V2P(a) (((uint) (a)) - KERNBASE)
#define P2V(a) ((void *)(((char *) (a)) + KERNBASE))
```

- マッピングの図からわかるように xv6 のメモリ管理では 2 GB 以上の物理メモリを認識できない

---

### xv6 の fork 時のページテーブル用意

- fork 処理は `proc.c` の`fork` 関数で実装されている
  - **fork はカーネルによる処理**
- ページテーブルに関連する処理は `vm.c` の `copyuvm` 関数で行われる
  - まず `setupkvm` 関数でカーネルアドレス空間のマッピングを用意
  - その後ユーザアドレス空間のメモリを親プロセスからコピー
    - **JOS とは違い copy-on-write ではない**

```c
pde_t*
copyuvm(pde_t *pgdir, uint sz)
{
  pde_t *d;
  pte_t *pte;
  uint pa, i, flags;
  char *mem;

  if((d = setupkvm()) == 0)
    return 0;
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walkpgdir(pgdir, (void *) i, 0)) == 0)
      panic("copyuvm: pte should exist");
    if(!(*pte & PTE_P))
      panic("copyuvm: page not present");
    pa = PTE_ADDR(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto bad;
    memmove(mem, (char*)P2V(pa), PGSIZE);
    if(mappages(d, (void*)i, PGSIZE, V2P(mem), flags) < 0) {
      kfree(mem);
      goto bad;
    }
  }
  return d;

bad:
  freevm(d);
  return 0;
}
```

- カーネル内での処理なので PTE (page table entry) から読み出した物理アドレスを (カーネル領域内にマッピングされた) 仮想アドレスに変換してアクセス... といったことができる

---

### JOS のアドレス空間

- JOS の仮想アドレス空間レイアウトは `inc/memlayout.h` に存在する

```c
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 *
 */
```

- xv6 と同様 KERNBASE (0xf0000000) より上をカーネル用の領域にしている
  - ただしそのアドレスは違う

---

### JOS の fork 時のページテーブル用意

- **xv6 とは違い fork がユーザにライブラリとして提供されている**
  - カーネルをシンプルにできる
  - fork 処理内容をユーザが自分で選択、実装できる
  - ユーザプロセス空間でページテーブルを作成するために UVPT というマッピングが用意されている (後述)
- **xv6 とは違い copy-on-write**

#### UVPT

- 子プロセスのページテーブルを用意する際に問題となるのは親のページテーブル内容をどう確認するか
  - ページテーブルはカーネルで管理されているのでアクセスできない
- そのために UVPT というマッピングを予めカーネルは用意しておく
  - UVPT についての公式な説明は [こちら][1] にある

```c
    // at env_setup_vm in kern/env.c
    //
    // UVPT maps the env's own page table read-only.
    // Permissions: kernel R, user R
    //
    // PDX(addr) で仮想アドレス addr を解決するための PDE の index がわかる
    // PADDR(addr) でカーネル空間を指す仮想アドレス addr の物理アドレスがわかる
    // e->env_pgdir は作成中のプロセスのページディレクトリの仮想アドレス
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;
```

- CPU によるページテーブルを使用した仮想アドレスの解決
  - CR3 レジスタから page directory table の物理アドレスを得る
  - 上位 10 bit を page directory table のエントリ解決に使用、page table の物理アドレスを得る
  - 次の 10 bit を page table のエントリ解決に使用、page の物理アドレスを得る
  - 残りの 12 bit で page 内の offset を指定、データにアクセス

<figure>
  <img
    src="/images/xv6-jos-page-table/figure2.jpg"
    title="how to resolve virtual address"
    alt="how to resolve virtual address"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

- UVPT は「自分自身を参照する PDE」
  - page directory table や page table 自体にアクセスするために使用できる
- 例えば 1023 番目の page table にアクセスしたい場合、`0xef7ff000 + offset` の仮想アドレスを使用する

<figure>
  <img
    src="/images/xv6-jos-page-table/figure3.jpg"
    title="how to access page table using UVPT"
    alt="how to access page table using UVPT"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

- 例えば page directory table にアクセスしたい場合、`0xef7bd000 + offset` の仮想アドレスを使用する

<figure>
  <img
    src="/images/xv6-jos-page-table/figure4.jpg"
    title="how to access page directory table using UVPT"
    alt="how to access page directory table using UVPT"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

[1]: https://pdos.csail.mit.edu/6.828/2018/labs/lab4/uvpt.html
[2]: https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev10.pdf
[3]: {% post_url 2019-11-10-qemu-gdb %}
