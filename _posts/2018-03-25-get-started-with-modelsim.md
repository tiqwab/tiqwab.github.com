---
layout: post
title: ModelSim ことはじめ
tags: "modelsim, fpga"
comments: true
---

ModelSim というより正確には ModelSim - Intel FPGA Edition を使い始めるためのメモです。

環境

- OS: Windows 10
- ModelSim Intel FPGA Starter Edition: 10.5b

---

### ModelSim Intel FPGA Edition とは

[こちら][2] から引用させて頂きます。

> ModelSim\* - Intel® FPGA Edition は、メンターグラフィックス社の ModelSim ソフトウェアを、インテル® デバイスを限定対象とした製品で、設計した自分のデジタル論理回路が意図した通りの動作をしているかをソフトウェア上で確認するためのシミュレータ・ツールです。FPGA 開発では、作業工程のうち論理シミュレーション（”RTL レベル・シミュレーション” とも言います）で使用します。

現在 FPGA 開発に Quartus Prime Lite Edition を使用しているのですが、これまでは HDL ファイル書き換えるたびにコンパイル、コンフィギュレーションして実機 (FPGA ボード) で確認して... とやっていました。しかし一連の操作に多少時間がかかるので、ある程度の動作確認ならばシミュレーションで済ませてしまいたい、というのが ModelSim を使おうと思ったモチベーションです。

### ModelSim のインストール

- [ここ][1] からインストールした
  - 本来は Quartus Prime と一緒にインストールすればよかった
  - 以前インストール時はチェックボックス選択しなかったので、今回 ModelSim だけ落とした

### ModelSim 付属のサンプル実行

- [ModelSimのGUIを使った基本シミュレーション][6] に則って ModelSim 付属の basicSimulation サンプルを実行してみる
  - このあとの操作を行うと色々ディレクトリ内にファイルが生成されるので、basicSimulation 以下をコピーしてそちらを使用する方がいいかも
  - 資料では VHDL を使用しているが Verilog HDL の場合もほぼ同様でできた
- ModelSim の起動
  - 実行ファイルのパスはインストール設定によるが、自分の場合は `C:\intelFPGA\17.1\modelsim_ase\win32aloem\vsim.exe` だった

### モジュールとテストベンチの作成

- テストベンチの記述については勉強中だが以下の資料を参考にしている
  - [わかる Verilog HDL 入門][4]
  - [はじめてみよう！テストベンチ Verilog HDL 編][5]

今回は ModelSim の操作確認のために簡単な 7 セグメントデコーダを書きました。

```verilog
module SEG7DEC(
    input [3:0] SW,
    output [6:0] HEX0
);

assign HEX0 = dec(SW);

function [6:0] dec(
    input [3:0] x
);
case (x)
    4'd0 : dec = 7'b1000000;
    4'd1 : dec = 7'b1111001;
    4'd2 : dec = 7'b0100100;
    4'd3 : dec = 7'b0110000;
    4'd4 : dec = 7'b0011001;
    4'd5 : dec = 7'b0010010;
    4'd6 : dec = 7'b0000010;
    4'd7 : dec = 7'b1011000;
    4'd8 : dec = 7'b0000000;
    4'd9 : dec = 7'b0010000;
    4'd10: dec = 7'b0001000;
    4'd11: dec = 7'b0000011;
    4'd12: dec = 7'b1000110;
    4'd13: dec = 7'b0100001;
    4'd14: dec = 7'b0000110;
    4'd15: dec = 7'b0001110;
    default: dec = 7'hxxxxxxx;
endcase
endfunction

endmodule

```

あとそのテストベンチです。

```verilog
module SEG7DEC_SIM();

reg [3:0] sw;
wire [6:0] hex0;

SEG7DEC seg7dec(sw, hex0);

initial begin
    sw = 4'b0;
end

initial begin
    $monitor("%t: %b", $time, sw);
end

initial begin
    #10
    sw = 4'b0001;
    #10
    sw = 4'b0011;
    #10
    sw = 4'b0111;
    #10
    sw = 4'b1111;
    #60
    sw = 4'b0000;
    #10
    $finish;
end

endmodule
```

### シミュレーション手順

- [ModelSim - Intel FPGA Edition - RTL シミュレーションの方法][7]
  - 色々自分で手順をまとめていたが、あとで見つけたこちらの方がわかりやすかった
- 既存のプロジェクトを開きたい場合は `[File] -> [Open]` でプロジェクトファイル (`*.mpf`) を指定すればよさそう

シミュレーション結果はこんな感じで、波形が見れたり monitor している内容が Transcript に表示されているのを確認できたりします。

<img
  src="/images/get-started-with-modelsim/wave.png"
  title="wave"
  alt="image of simulation by ModelSim"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

### 参考

- [はじめてみよう！テストベンチ \| マクニカオンラインサービス][3]

[1]: https://www.altera.co.jp/products/design-software/model---simulation/modelsim-altera-software.html
[2]: https://service.macnica.co.jp/library/110113
[3]: https://service.macnica.co.jp/library/110601
[4]: https://www.amazon.co.jp/%E3%82%8F%E3%81%8B%E3%82%8BVerilog-HDL%E5%85%A5%E9%96%80%E2%80%95%E6%96%87%E6%B3%95%E3%81%AE%E5%9F%BA%E7%A4%8E%E3%81%8B%E3%82%89%E8%AB%96%E7%90%86%E5%9B%9E%E8%B7%AF%E8%A8%AD%E8%A8%88%E3%80%81%E8%AB%96%E7%90%86%E5%90%88%E6%88%90%E3%80%81%E5%AE%9F%E8%A3%85%E3%81%BE%E3%81%A7-%E3%83%88%E3%83%A9%E3%83%B3%E3%82%B8%E3%82%B9%E3%82%BF%E6%8A%80%E8%A1%93SPECIAL-%E6%9C%A8%E6%9D%91-%E7%9C%9F%E4%B9%9F/dp/4789837564
[5]: https://service.macnica.co.jp/library/110605
[6]: http://www.paltek.co.jp/mentorkh/01/PALTEK-1-ModelSim_GUI.pdf
[7]: https://service.macnica.co.jp/library/118733
