---
layout: post
title: HariboteOS で気付いた BIOS ブートの勘違い
tags: "bios, mbr, hariboteos"
comments: true
---

[30 日 OS 自作入門][1] の 1, 2 日目を読みながら 「BIOS ならディスクの先頭セクタって MBR があるんじゃないの?」 と疑問に思っていたのですが、それは自分の BIOS についての勘違いだということが整理できたのでメモしておこうと思います。

---

#### これまでの勘違い

BIOS 環境の PC で OS を起動する場合ブートに使用するディスクはパーティションテーブル (ほとんどの場合 MBR) で管理されている。

#### 実際は

BIOS 環境の PC で OS を起動するのにパーティションテーブルは必須ではない。

---

[30 日 OS 自作入門][1] や [ブートセクタ][2] に書かれていますが、BIOS は指定されたブロックデバイスの先頭セクタをメモリ番地 0x0000:0x7c00 に読み込みそこに jmp して実行するだけというようです。なので [MBR][4] が存在するかとか各パーティションのブート可能フラグがどうかといった話は BIOS さんには全く関係のない話なんですね。

もちろん現実に OS を利用する場合その PC につながるブロックデバイスは何らかのパーティションをされていたりフォーマットされていたりすると思いますが、それはデータをファイルという形で便利に扱いたいとかそういうためのもので、(OS の定義による気がしますが) それがもし必要ないというのであれば一切パーティションやフォーマットをしていないディスクであっても OS のブートストラップや実行に不都合は無いんだと思います。

[30 日 OS 自作入門][1] だと FAT12 ファイルシステムとして認識してもらえるようにフロッピーディスクイメージを作成していますが、これが上手くいくのは FAT12 の予約領域にはブートストラッププログラムを書けるようなスペースがあり、またディスク先頭 3 bytes にそこへ飛ぶための JMP 命令が書けるので、結果として BIOS で先頭セクタを読んでも上手くいくという感じみたいです。FAT12 については以下のあたりの資料が参考に成りました。

- [FAT12,FAT16の構造(予約領域編)][5]
- [0から作るOS開発 ブートローダその１１ FAT12ファイルシステムからファイルを読み込む][6]

ということでパーティション等を一切気にしなくても Hello World を出力したいだけなら先頭セクタに以下のコード (をアセンブルしたもの) をベタッと置けばそれがそのまま BIOS に起動してもらえます。

```asm
  ORG 0x7c00

entry:
  MOV AX,0
  MOV SS,AX
  MOV SP,0x7c00
  MOV DS,AX
  MOV ES,AX
  MOV SI,msg ; load an address of msg

putloop: ; print character one by one
  MOV AL,[SI]
  ADD SI,1
  CMP AL,0
  JE fin
  MOV AH,0x0e
  MOV BX,15
  INT 0x10
  JMP putloop

fin:
  HLT
  JMP fin

msg:
  DB 0x0a, 0x0a
  DB "Hello World"
  DB 0x0a
  DB 0

  TIMES 0x1fe-($-$$) DB 0 ; (0x7dfe + 2) - 0x7c00 = 0x200 = 512 bytes. 2 is the below DB bytes.
  DB 0x55, 0xaa ; the end of boot sector must be 0x55, 0xaa
```

(Makefile 等全体は [github リポジトリ][3] に配置しました)

---

ちなみに現在の PC だと BIOS というより UEFI を利用する環境の方が多いと思いますが、こちらであれば通常は GPT というパーティションテーブルを利用して実行するアプリケーションを探しに行くのでパーティションテーブル が必須と言っていいと思います。

[1]: https://www.amazon.co.jp/dp/B00IR1HYI0/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1
[2]: https://ja.wikipedia.org/wiki/%E3%83%96%E3%83%BC%E3%83%88%E3%82%BB%E3%82%AF%E3%82%BF
[3]: https://github.com/tiqwab/example/tree/master/hello-without-fat
[4]: https://ja.wikipedia.org/wiki/%E3%83%9E%E3%82%B9%E3%82%BF%E3%83%BC%E3%83%96%E3%83%BC%E3%83%88%E3%83%AC%E3%82%B3%E3%83%BC%E3%83%89
[5]: https://qiita.com/iria_piyo/items/d949b93bc056a8c370d6
[6]: http://softwaretechnique.jp/OS_Development/bootloader11.html
