---
layout: post
title: RSA 暗号を実装する
tags: "rsa,ctf,cryptography"
comments: true
---

RSA 暗号といえば古くから存在する公開鍵暗号アルゴリズムですが、ふとしたきっかけから自分で実装してみたので、そのときに参照した RFC や調べたことをまとめました。
実装したコードは [こちら][7] にあるので、以降の内容と併せて読んでもらえればと思います。

---

### RSA について

RSA 暗号の鍵ペア、暗号化、復号はそれぞれ以下のように表されます。

#### 鍵ペア

\\( p, q \\) を異なる素数として、\\( n := pq \\) とする。整数 \\( e \\) を選び、整数 \\( d \\) を以下の式を満たすように取る。

\\( de \\equiv 1 \\mod (p - 1)(q - 1) \\)

このとき \\( (n, e) \\) を公開鍵、 \\( d \\) を秘密鍵とする。

(以降は n, e, d をこの定義と同じ意味で使用します)

#### 暗号化

平文 \\( m \\) から暗号文 \\( c \\) を \\( c = m^e \\bmod n \\) で生成する。

#### 復号

暗号文 \\( c \\) から元の平文 \\( m \\) を \\( m = c^d \\bmod n\\) で得る。

### RSA 暗号化、復号 with OpenSSL

自分で RSA を実装する前に、OpenSSL を使用して鍵生成、暗号化、復号を試してみます。
なおここではあとで内容を確認したいということもあり、鍵長 1024 bit という短いものを使用しています。

```
# 1024 bit 長の RSA 秘密鍵生成
$ openssl genrsa 1024 > test.1024.key

# 秘密鍵の内容確認
$ openssl rsa -text < test.1024.key

# 公開鍵生成
$ openssl rsa -pubout < test.1024.key > test.1024.pub

# 公開鍵の内容確認
$ openssl rsa -text -pubin -in test.1024.pub

# 暗号化
$ echo "hello world" | openssl pkeyutl -encrypt -pubin -inkey test.1024.pub > encrypted

# 復号
$ openssl pkeyutl -decrypt -inkey test.1024.key < encrypted
hello world
```

暗号化、復号で後述の OAEP を使う場合は以下のようなコマンドになります。

```
# 暗号化
$ echo "hello world" | openssl pkeyutl -encrypt -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256 -pubin -inkey test.1024.pub > encrypted

# 復号
$ openssl pkeyutl -decrypt -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256 -inkey test.1024.key < encrypted
hello world
```

### RSA 公開鍵の内容確認

先程 OpenSSL で生成した公開鍵の内容を openssl コマンドで確認します。
RSA 公開鍵なので n (Modulus), e (Exponent) の値が含まれていることがわかります。

```
$ openssl rsa -text -pubin -in test.1024.pub
Public-Key: (1024 bit)
Modulus:
    00:a7:6a:bf:b3:10:2f:6c:e8:ca:da:5e:aa:6a:bb:
    60:ad:14:fb:af:a0:2e:48:7a:57:0b:70:7a:97:19:
    a2:1b:79:f9:cb:89:f3:96:86:6d:65:18:92:d3:3f:
    05:19:bf:a7:63:ff:9d:35:b0:f5:7b:99:f2:ad:f8:
    88:e2:b1:ca:36:cb:3a:12:b2:da:b2:5c:5d:7c:e9:
    e8:0e:bd:b7:29:fb:78:a6:e1:98:c9:bd:68:55:90:
    f3:c2:4d:fa:80:25:70:42:9d:84:98:fc:f9:c8:f7:
    25:ae:c9:f2:e2:b2:87:66:ae:15:5e:ff:cb:99:43:
    6f:84:6a:5c:c4:b5:43:2d:15
Exponent: 65537 (0x10001)
```

余談ですが OpenSSL での鍵生成では e の値として 3 or 65537 しか選択肢がなくデフォルトは 65537 となります。
e として 65537 を選ぶ根拠を深くは理解できていませんが、適当なものを選ばないと攻撃の糸口に使われる可能性があるようです (参考: [RSA 暗号を理解して有名な攻撃を試してみる][11])。



公開鍵のフォーマットを理解するために、このファイルから人力で n, e を読み取ってみます。

まず公開鍵の内容を見ると以下のようなテキストファイルであることがわかります。
このフォーマットは PEM と呼ばれており、以下の要素で構成されます。

- 開始ラベル (今回だと BEGIN PUBLIC KEY を含む行)
- Base64 エンコーディングされた文字列
- 終了ラベル (今回だと END PUBLIC KEY を含む行)

```
$ cat test.1024.pub
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCnar+zEC9s6MraXqpqu2CtFPuv
oC5IelcLcHqXGaIbefnLifOWhm1lGJLTPwUZv6dj/501sPV7mfKt+Ijisco2yzoS
stqyXF186egOvbcp+3im4ZjJvWhVkPPCTfqAJXBCnYSY/PnI9yWuyfLisodmrhVe
/8uZQ2+EalzEtUMtFQIDAQAB
-----END PUBLIC KEY-----
```

鍵や証明書のエンコーディングで PEM が使用される場合は、[RFC 7468][1] でラベルとその内容の紐付けが仕様化されており、BEGIN PUBLIC KEY ラベルを持つデータの内容は Subject Public Key Info (SPKI) であるとされています。

SPKI というのは X.509 証明書に含まれる項目の一つであり、RFC だと [RFC 5280][2] で定義されます。公開鍵に関する仕様というと何となく PKCS に含まれていそうなものですが、公開鍵全般に使えるようなフォーマットの仕様は無く、この SPKI が一般に使われているようです (ちなみに RSA に限定すると PKCS #1 がある)。

RFC 5280 では SPKI のフォーマットが、 ASN.1 というデータ構造を定義するための記法を用いて定義されています。
ASN.1 やそのシリアライズ方式である DER が何かについてはここではあまり扱いませんが、Let's Encrypt の [ASN.1 と DER へようこそ][3] は概要を理解するのに参考になると思います。

SPKI の ASN.1 定義は algorithm と subjectPublicKey の二つから成ります。

```
SubjectPublicKeyInfo  ::=  SEQUENCE  {
     algorithm            AlgorithmIdentifier,
     subjectPublicKey     BIT STRING  }
```

algorithm は OID により暗号方式を指定しており、RSA の場合は rsaEncryption を示す OID 1.2.840.113549.1.1.1 の固定値です。各種公開鍵暗号アルゴリズムとその OID は [RFC 3279][5] の仕様から見つけられます。

subjectPublicKey は実際の鍵が含まれる箇所であり、RSA の場合は PKCS #1 ([RFC 8017][6]) の RSAPublicKey のデータ構造が含まれます。

```
RSAPublicKey ::= SEQUENCE {
    modulus           INTEGER,  -- n
    publicExponent    INTEGER   -- e
}
```

以上で公開鍵ファイルから n, e を読み取る方法がわかったので、実際に Base64 デコードした公開鍵の内容を確認してみます。
なおここでは理解のために人力でデコードしますが、単純に内容確認するなら [ASN.1 JavaScript decoder][4] を使うのが便利そうです。

```
 000000 30 81 9f 30 0d 06 09 2a 86 48 86 f7 0d 01 01 01  >0..0...*.H......<
 000010 05 00 03 81 8d 00 30 81 89 02 81 81 00 a7 6a bf  >......0.......j.<
 000020 b3 10 2f 6c e8 ca da 5e aa 6a bb 60 ad 14 fb af  >../l...^.j.`....<
 000030 a0 2e 48 7a 57 0b 70 7a 97 19 a2 1b 79 f9 cb 89  >..HzW.pz....y...<
 000040 f3 96 86 6d 65 18 92 d3 3f 05 19 bf a7 63 ff 9d  >...me...?....c..<
 000050 35 b0 f5 7b 99 f2 ad f8 88 e2 b1 ca 36 cb 3a 12  >5..{........6.:.<
 000060 b2 da b2 5c 5d 7c e9 e8 0e bd b7 29 fb 78 a6 e1  >...\]|.....).x..<
 000070 98 c9 bd 68 55 90 f3 c2 4d fa 80 25 70 42 9d 84  >...hU...M..%pB..<
 000080 98 fc f9 c8 f7 25 ae c9 f2 e2 b2 87 66 ae 15 5e  >.....%......f..^<
 000090 ff cb 99 43 6f 84 6a 5c c4 b5 43 2d 15 02 03 01  >...Co.j\..C-....<
 0000a0 00 01                                            >..<
 0000a2
```

- 0x00 からの `30 81 9f` で長さ 0x9f の SEQUENCE (SubjectPublicKeyInfo)
  - 0x03 からの `30 0d` で長さ 0x0d の SEQUENCE (algorithm)
    - 0x05 からの `06 09 2a 86 48 86 f7 0d 01 01 01` で OID 1.2.840.113549.1.1.1 の rsaEncryption を指す
    - 0x10 からの `05 00` は NULL
  - 0x012 からの `03 81 8d 00` は長さ 0x8d の BIT STRING (subjectPublicKey)
    - 今回は中身は PKCS#1 の公開鍵になるのでこのまま読み進めてみる
    - 0x16 からの `30 81 89` で長さ 0x89 の SEQUECNE (RSAPublicKey)
      - 0x19 からの `02 81 81` で長さ 0x81 の INTEGER -> ここからが n (modulus)
      - 0x9d からの `02 03` で長さ 0x03 の INTEGER -> ここからが e (publicExponent) で値は 0x010001 (65537)

n や e の値がはじめに openssl で確認した値と一致しており、正しく読み取れたことがわかります。
以上を踏まえてサンプルコードでは公開鍵の読み込みを `read_rsa_pub_key` 関数で実装しています。

### RSA 暗号化の実装

RSA の暗号化を実装するのに必要な情報は [RFC 8017][6] (PKCS #1) に含まれます。
暗号化の手順は 7. Encryption Schemes に記述があり、以下のようになります。

- 暗号化スキームにより平文をエンコード
- エンコードしたバイト列を整数値に変換
- RSA 暗号化
- 暗号化の出力値をバイト列に変換

#### 暗号化スキームにより平文をエンコード

RSA 暗号化の最初のステップでは、平文を暗号化スキームにより鍵長と同じ k バイトのバイト列に変換します。
直接平文を RSA 暗号の入力としないのは、より安全な暗号を生成するためです。
暗号化スキームがどのように安全さに貢献しているかは全体を理解できてはいないのですが、わかりやすいところでいうとアルゴリズム全体を非決定的にするというものがあります。RSA 暗号自体は決定的なアルゴリズムなので同じ入力には同じ出力が返されるのですが、暗号化スキームとして乱数を含めるステップがあり、それにより全体としては非決定的なアルゴリズムにできます。

RFC 中では以下の 2 つの暗号化スキームが紹介されています。

- RSAES-OAEP
- RSAES-PKCS1-v1\_5

OpenSSL がデフォルト採用するのは RSAES-PKCS1-v1\_5 なのですが、RFC や [電子政府における調達のために参照すべき暗号のリスト][8] を見ると、2023 年現在では RSAES-PKCS1-v1\_5 は互換性維持目的でのみ採用が認められており、推奨は RSAES-OAEP という扱いになっています。これに従い、ここでは RSAES-OAEP を使用することにします。

RSAES-OAEP の実装は RFC 中の 7.1.1 Encryption Operation で詳細に定義されており、サンプルコードでは `oaep_encode_sha256` 関数として実装しています。
RSAES-OAEP 実装ではどのハッシュ関数やマスク生成関数 (MFG) を使用するかの選択があるのですが、サンプルコードではそれぞれ SHA-256 と MGF1 (RFC 中のAppendix B) で決め打ちしています。これらの情報は暗号文には載らないので、運用上は暗号化する主体と復号する主体の間で別途合意するのだと思います。

#### エンコードしたバイト列を整数値に変換

RSAES-OAEP でエンコードしたバイト列を RSA 暗号で扱うために整数に変換します。
RFC 中では 4.1 OS2IP で手順が定義されており、いわばバイト列を big endian な 256 進数の整数として扱うような考えになっています。
サンプルコードでは `OS2IP` 関数として実装しています。

#### RSA 暗号化

ここでようやく RSA 暗号化を行います。
前ステップの出力を m として \\( m^e \\bmod n \\) が暗号結果です。
サンプルコードでは `RSAEP` 関数として実装しています。
実装では [binary method][9] と呼ばれるような計算方法を取ることで、計算量を \\( O(e) \\) から \\( O(\\log(n)) \\) に落としています。

#### 暗号化の出力値をバイト列に変換

最後に RSA 暗号化の出力値である整数をバイト列に変換して暗号化全体の出力とします。
RFC 中では 4.2 I2OSP で操作が定義されており、内容的には先程見た OS2IP の逆を行うような手順です。
サンプルコードでは `I2OSP` 関数として実装しています。

以上で RSA 暗号化の実装ができたので、OpenSSL との間で動作を確認してみます。

```
$ cat plain.text
This is plain text.

# 自前の RSA 暗号化
$ python3 rsa.py encrypt test.1024.pub plain.text encrypted

# OpenSSL による復号
$ openssl pkeyutl -decrypt -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256 -inkey test.1024.key < encrypted
This is plain text.
```

問題無さそうです。

続いて RSA 復号側の実装に入ります。

### RSA 秘密鍵の内容確認

RSA 秘密鍵から d を取り出すために、秘密鍵のフォーマットを確認します。
OpenSSL で生成した秘密鍵には以下の内容が含まれます。

```
$ openssl rsa -text -in test.1024.key
Private-Key: (1024 bit, 2 primes)
modulus:
    00:a7:6a:bf:b3:10:2f:6c:e8:ca:da:5e:aa:6a:bb:
    60:ad:14:fb:af:a0:2e:48:7a:57:0b:70:7a:97:19:
    a2:1b:79:f9:cb:89:f3:96:86:6d:65:18:92:d3:3f:
    05:19:bf:a7:63:ff:9d:35:b0:f5:7b:99:f2:ad:f8:
    88:e2:b1:ca:36:cb:3a:12:b2:da:b2:5c:5d:7c:e9:
    e8:0e:bd:b7:29:fb:78:a6:e1:98:c9:bd:68:55:90:
    f3:c2:4d:fa:80:25:70:42:9d:84:98:fc:f9:c8:f7:
    25:ae:c9:f2:e2:b2:87:66:ae:15:5e:ff:cb:99:43:
    6f:84:6a:5c:c4:b5:43:2d:15
publicExponent: 65537 (0x10001)
privateExponent:
    51:f3:c6:6d:50:21:f7:0d:29:a7:a5:a9:84:5f:bf:
    1e:5a:e4:2d:7f:9a:c8:6d:e2:c8:3d:c2:34:cf:1e:
    74:96:cb:f9:9f:c8:f6:c9:4d:29:ac:d2:ca:c7:d1:
    a6:5e:14:01:b6:71:ed:83:77:57:8e:ef:a5:cb:c0:
    ae:3f:db:bd:16:23:0c:28:6b:be:82:f3:30:06:c2:
    4f:22:a8:cc:22:d4:f1:58:09:32:8c:62:75:8b:97:
    9c:4d:46:dc:63:f5:bf:0d:ec:da:41:4e:c3:61:a1:
    e7:6d:ef:31:6d:7a:7b:cd:d3:65:4c:58:03:20:42:
    07:b3:fe:a9:36:08:4d:01
prime1:
    00:d1:1e:62:a3:c1:5b:79:68:fe:06:85:8e:a6:d8:
    bf:8c:79:c9:f9:33:e8:a7:d1:e3:b7:d1:8d:73:5c:
    ee:65:b1:e5:de:31:fc:de:5a:ad:94:3c:78:bc:34:
    2b:66:94:a1:89:fe:46:c9:8e:7d:d0:18:1d:10:81:
    72:b9:fa:ea:41
prime2:
    00:cc:f3:0b:54:05:74:4b:67:b3:42:94:c4:15:e1:
    3e:03:52:cf:1c:62:d9:03:19:31:be:fb:89:85:43:
    23:0b:f1:0e:cf:ae:99:7c:29:1c:b5:36:5e:0a:b1:
    15:67:b3:e9:33:00:bb:f3:94:b8:c1:14:53:68:c6:
    aa:b8:2b:05:d5
exponent1:
    00:8b:ef:48:54:8a:5c:3a:f7:5e:1d:61:1c:1f:5c:
    25:79:cc:39:b2:8f:e0:dd:04:1f:dc:ee:d6:37:df:
    75:0c:0a:2a:67:30:8e:25:01:0a:ec:8a:36:c4:c2:
    28:54:c1:9b:03:6b:6b:55:0f:0b:f3:c7:5f:13:9f:
    7b:f5:26:09:c1
exponent2:
    00:a8:70:5f:b1:10:42:81:ee:9a:6f:70:20:af:f2:
    cc:aa:a2:96:41:38:24:2e:dd:b7:fa:c4:74:43:a7:
    e7:d7:da:a8:57:9b:a1:dd:5f:54:8e:c2:3e:0b:ff:
    7a:1e:1e:c8:db:f8:10:80:a2:8c:2d:73:6d:11:c1:
    a5:71:73:3a:79
coefficient:
    00:b1:7f:48:b0:51:05:66:fd:24:dd:46:8e:2b:ca:
    38:6a:6b:aa:53:56:fc:34:b5:5a:c8:e0:4b:63:de:
    23:29:2e:fb:f0:28:a7:1e:90:d0:bd:92:e8:aa:8f:
    b1:79:25:92:9f:64:78:51:f7:27:a9:69:98:5f:3b:
    ac:ef:ea:cc:58
```

[RFC 8017][6] の 5.1.2 RSADP 項を読むと、復号にはいずれかの情報の組があれば良いようです。

- (modulus, privateExponent)
- (prime1, prime2, exponent1, exponent2, coefficient)

以降では n = modulus, d = privateExponent を復号に使用します。n は公開鍵の一部でもあるので、実際に秘密鍵だけが持つ情報は d です。

秘密鍵は公開鍵同様 PEM フォーマットで出力されています。
公開鍵でも PEM の仕様として参照した [RFC 7468][1] から、このファイルが PKCS #8 の PrivateKeyInfo (OneAsymmetric Key) の内容を含むことがわかります。

```
-----BEGIN PRIVATE KEY-----
MIICeAIBADANBgkqhkiG9w0BAQEFAASCAmIwggJeAgEAAoGBAKdqv7MQL2zoytpe
qmq7YK0U+6+gLkh6VwtwepcZoht5+cuJ85aGbWUYktM/BRm/p2P/nTWw9XuZ8q34
iOKxyjbLOhKy2rJcXXzp6A69tyn7eKbhmMm9aFWQ88JN+oAlcEKdhJj8+cj3Ja7J
8uKyh2auFV7/y5lDb4RqXMS1Qy0VAgMBAAECgYBR88ZtUCH3DSmnpamEX78eWuQt
f5rIbeLIPcI0zx50lsv5n8j2yU0prNLKx9GmXhQBtnHtg3dXju+ly8CuP9u9FiMM
KGu+gvMwBsJPIqjMItTxWAkyjGJ1i5ecTUbcY/W/DezaQU7DYaHnbe8xbXp7zdNl
TFgDIEIHs/6pNghNAQJBANEeYqPBW3lo/gaFjqbYv4x5yfkz6KfR47fRjXNc7mWx
5d4x/N5arZQ8eLw0K2aUoYn+RsmOfdAYHRCBcrn66kECQQDM8wtUBXRLZ7NClMQV
4T4DUs8cYtkDGTG++4mFQyML8Q7Prpl8KRy1Nl4KsRVns+kzALvzlLjBFFNoxqq4
KwXVAkEAi+9IVIpcOvdeHWEcH1wlecw5so/g3QQf3O7WN991DAoqZzCOJQEK7Io2
xMIoVMGbA2trVQ8L88dfE5979SYJwQJBAKhwX7EQQoHumm9wIK/yzKqilkE4JC7d
t/rEdEOn59faqFebod1fVI7CPgv/eh4eyNv4EICijC1zbRHBpXFzOnkCQQCxf0iw
UQVm/STdRo4ryjhqa6pTVvw0tVrI4Etj3iMpLvvwKKcekNC9kuiqj7F5JZKfZHhR
9yepaZhfO6zv6sxY
-----END PRIVATE KEY-----
```

PKCS #8 ([RFC 5208][10]) は (RSA によらない) 秘密鍵の構文についての仕様であり、PrivateKeyInfo の ASN.1 が以下のように定義されています。

```
PrivateKeyInfo ::= SEQUENCE {
  version                   Version,
  privateKeyAlgorithm       PrivateKeyAlgorithmIdentifier,
  privateKey                PrivateKey,
  attributes           [0]  IMPLICIT Attributes OPTIONAL }

Version ::= INTEGER

PrivateKeyAlgorithmIdentifier ::= AlgorithmIdentifier

PrivateKey ::= OCTET STRING

Attributes ::= SET OF Attribute
```

このうち、PrivateKey の内容は、今回 [RFC 8017][6] で定義される PKCS #1 フォーマットなので、以下の情報が含まれます。

```
RSAPrivateKey ::= SEQUENCE {
    version           Version,
    modulus           INTEGER,  -- n
    publicExponent    INTEGER,  -- e
    privateExponent   INTEGER,  -- d
    prime1            INTEGER,  -- p
    prime2            INTEGER,  -- q
    exponent1         INTEGER,  -- d mod (p-1)
    exponent2         INTEGER,  -- d mod (q-1)
    coefficient       INTEGER,  -- (inverse of q) mod p
    otherPrimeInfos   OtherPrimeInfos OPTIONAL
}
```

以上の内容を踏まえ、実際に秘密鍵 PEM ファイルの Base64 エンコーディングされた部分の内容を見てみます。
公開鍵の方でも触れましたが、サクッと内容確認するなら [ASN.1 JavaScript decoder][4] が便利です。

```
000000 30 82 02 78 02 01 00 30 0d 06 09 2a 86 48 86 f7  >0..x...0...*.H..<
000010 0d 01 01 01 05 00 04 82 02 62 30 82 02 5e 02 01  >.........b0..^..<
000020 00 02 81 81 00 a7 6a bf b3 10 2f 6c e8 ca da 5e  >......j.../l...^<
000030 aa 6a bb 60 ad 14 fb af a0 2e 48 7a 57 0b 70 7a  >.j.`......HzW.pz<
000040 97 19 a2 1b 79 f9 cb 89 f3 96 86 6d 65 18 92 d3  >....y......me...<
000050 3f 05 19 bf a7 63 ff 9d 35 b0 f5 7b 99 f2 ad f8  >?....c..5..{....<
000060 88 e2 b1 ca 36 cb 3a 12 b2 da b2 5c 5d 7c e9 e8  >....6.:....\]|..<
000070 0e bd b7 29 fb 78 a6 e1 98 c9 bd 68 55 90 f3 c2  >...).x.....hU...<
000080 4d fa 80 25 70 42 9d 84 98 fc f9 c8 f7 25 ae c9  >M..%pB.......%..<
000090 f2 e2 b2 87 66 ae 15 5e ff cb 99 43 6f 84 6a 5c  >....f..^...Co.j\<
0000a0 c4 b5 43 2d 15 02 03 01 00 01 02 81 80 51 f3 c6  >..C-.........Q..<
0000b0 6d 50 21 f7 0d 29 a7 a5 a9 84 5f bf 1e 5a e4 2d  >mP!..)...._..Z.-<
0000c0 7f 9a c8 6d e2 c8 3d c2 34 cf 1e 74 96 cb f9 9f  >...m..=.4..t....<
0000d0 c8 f6 c9 4d 29 ac d2 ca c7 d1 a6 5e 14 01 b6 71  >...M)......^...q<
0000e0 ed 83 77 57 8e ef a5 cb c0 ae 3f db bd 16 23 0c  >..wW......?...#.<
0000f0 28 6b be 82 f3 30 06 c2 4f 22 a8 cc 22 d4 f1 58  >(k...0..O".."..X<
000100 09 32 8c 62 75 8b 97 9c 4d 46 dc 63 f5 bf 0d ec  >.2.bu...MF.c....<
000110 da 41 4e c3 61 a1 e7 6d ef 31 6d 7a 7b cd d3 65  >.AN.a..m.1mz{..e<
000120 4c 58 03 20 42 07 b3 fe a9 36 08 4d 01 02 41 00  >LX. B....6.M..A.<
000130 d1 1e 62 a3 c1 5b 79 68 fe 06 85 8e a6 d8 bf 8c  >..b..[yh........<
000140 79 c9 f9 33 e8 a7 d1 e3 b7 d1 8d 73 5c ee 65 b1  >y..3.......s\.e.<
000150 e5 de 31 fc de 5a ad 94 3c 78 bc 34 2b 66 94 a1  >..1..Z..<x.4+f..<
000160 89 fe 46 c9 8e 7d d0 18 1d 10 81 72 b9 fa ea 41  >..F..}.....r...A<
000170 02 41 00 cc f3 0b 54 05 74 4b 67 b3 42 94 c4 15  >.A....T.tKg.B...<
000180 e1 3e 03 52 cf 1c 62 d9 03 19 31 be fb 89 85 43  >.>.R..b...1....C<
000190 23 0b f1 0e cf ae 99 7c 29 1c b5 36 5e 0a b1 15  >#......|)..6^...<
0001a0 67 b3 e9 33 00 bb f3 94 b8 c1 14 53 68 c6 aa b8  >g..3.......Sh...<
0001b0 2b 05 d5 02 41 00 8b ef 48 54 8a 5c 3a f7 5e 1d  >+...A...HT.\:.^.<
0001c0 61 1c 1f 5c 25 79 cc 39 b2 8f e0 dd 04 1f dc ee  >a..\%y.9........<
0001d0 d6 37 df 75 0c 0a 2a 67 30 8e 25 01 0a ec 8a 36  >.7.u..*g0.%....6<
0001e0 c4 c2 28 54 c1 9b 03 6b 6b 55 0f 0b f3 c7 5f 13  >..(T...kkU...._.<
0001f0 9f 7b f5 26 09 c1 02 41 00 a8 70 5f b1 10 42 81  >.{.&...A..p_..B.<
000200 ee 9a 6f 70 20 af f2 cc aa a2 96 41 38 24 2e dd  >..op ......A8$..<
000210 b7 fa c4 74 43 a7 e7 d7 da a8 57 9b a1 dd 5f 54  >...tC.....W..._T<
000220 8e c2 3e 0b ff 7a 1e 1e c8 db f8 10 80 a2 8c 2d  >..>..z.........-<
000230 73 6d 11 c1 a5 71 73 3a 79 02 41 00 b1 7f 48 b0  >sm...qs:y.A...H.<
000240 51 05 66 fd 24 dd 46 8e 2b ca 38 6a 6b aa 53 56  >Q.f.$.F.+.8jk.SV<
000250 fc 34 b5 5a c8 e0 4b 63 de 23 29 2e fb f0 28 a7  >.4.Z..Kc.#)...(.<
000260 1e 90 d0 bd 92 e8 aa 8f b1 79 25 92 9f 64 78 51  >.........y%..dxQ<
000270 f7 27 a9 69 98 5f 3b ac ef ea cc 58              >.'.i._;....X<
00027c
```

-  0x00 からの `30 82 02 78` で長さ 632 の SEQUENCE (PrivateKeyInfo)
  - 0x04 からの `02 01 00` で長さ 1 の INTEGER (version) で値は 0
  - 0x07 からの `30 0d 06 09 2a 86 48 86 f7 0d 01 01 01 05 00` が長さ 0x0d の SEQUENCE (privateKeyAlgorithm)
    - 0x05 からの `06 09 2a 86 48 86 f7 0d 01 01 01` で OID 1.2.840.113549.1.1.1 の rsaEncryption を指す
    - 0x10 からの `05 00` は NULL
  - 0x16 からの `04 82 02 62 ...` で長さ 610 の OCTET STRING (privateKey)
    - (ここに PKCS #1 フォーマットの private key が含まれる)
    - 0x1a からの `30 82 02 5e` で長さ 606 の SEQUENCE (RSAPrivateKey)
      - 0x1e からの `02 01 00` で長さ 1 の INTEGER (version) で値は 0
        - (一般的には 0、複数素数を使う RSA だと 1 になるとのこと)
      - 0x21 からの `02 81 81` で長さ 0x81 の INTEGER (n)
      - 0xa5 からの `02 03 01 00 01` で長さ 0x03 の INTEGER (e) で値は 0x010001 (65537)
      - 0xaa からの `02 81 80` で長さ 0x80 の INTEGER (d)
      - (... 以降も同様に読んでいけるはずだが省略)

というように人力でも n, e, d の値が読み取れ、OpenSSL で確認した秘密鍵の内容と一致していることが確認できました。
サンプルコードでは以上の内容を `read_rsa_private_key` 関数として実装しています。

### RSA 復号の実装

復号の仕様は [RFC 8017][6] の 7.1.2 Decryption Operation でまとめられており、その手順は以下のようになります。
暗号化を逆順に適用したような操作になっているため、実装的にも理解としても暗号化の操作から流用できる部分が多いです。

- 入力を整数値に変換
- RSA 復号
- 復号結果をバイト列に変換
- 暗号化スキームにより平文へデコード

#### 入力を整数値に変換

入力である暗号文を暗号化でも使用した OS2IP により整数値に変換します。

#### RSA 復号

\\( c^d \\bmod n \\) を計算して復号結果とします。
サンプルコードでは `RSADP` 関数で実装しています。

#### 復号結果をバイト列に変換

復号結果の整数値を I2OSP によりバイト列に変換します。

#### 暗号化スキームにより平文へデコード

RSAES-OAEP のデコード処理を行います。
仕様に従って実装すれば良く、サンプルコードでは `oaep_decode_sha256` 関数で実装しています。

---

最後に本文で参照した RFC をまとめておきます。

- [RFC 8017 PKCS #1: RSA Cryptography Specification Version 2.2][6]
  - RSA 実装について
  - RSA の暗号化、復号の手順や公開鍵、秘密鍵のフォーマットが含まれる
- [RFC 7468 Textual Encodings of PKIX, PKCS, and CMS Structures][1]
  - 鍵や証明書のテキストエンコーディングとして使われる PEM について
  - OpenSSL で生成した公開鍵や秘密鍵は PEM ファイルで出力される
- [RFC 5208 Public-Key Cryptography Standards (PKCS) #8: Private-Key Information Syntax Specification Version 1.2][10]
  - 秘密鍵の構文について
  - OpenSSL で生成した秘密鍵は PKCS #8 フォーマットを持つ (そしてそれが PEM エンコーディングされていた)
    - PKCS #8 に含まれる実際の RSA 秘密鍵のフォーマットが RFC 8017 の PKCS #1 フォーマットだった
- [RFC 3279 Algorithms and Identifiers for the Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile][5]
  - 本記事では各種公開鍵暗号を示す OID のまとめ先として参照した

[1]: https://www.rfc-editor.org/rfc/rfc7468
[2]: https://www.rfc-editor.org/rfc/rfc5280
[3]: https://letsencrypt.org/ja/docs/a-warm-welcome-to-asn1-and-der
[4]: https://lapo.it/asn1js/
[5]: https://www.rfc-editor.org/rfc/rfc3279
[6]: https://www.rfc-editor.org/rfc/rfc8017
[7]: https://gist.github.com/tiqwab/3fc0cba339940c6687f2b05f060135fb
[8]: https://www.cryptrec.go.jp/list.html 
[9]: https://en.wikipedia.org/wiki/Modular_exponentiation
[10]: https://www.rfc-editor.org/rfc/rfc5208
[11]: https://zenn.dev/anko/articles/ctf-crypto-rsa
