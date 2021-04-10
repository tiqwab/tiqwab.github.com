---
layout: post
title: BCP 47 と u-extension による言語表記
tags: "lang, go"
comments: true
---

多言語対応の必要なシステムやプログラムでは一般的に言語指定として `en` や `ja-JP` のような表記が使われています。これは IETF の [BCP 47][3] という仕様 (というか Best Current Practices なので慣習?) で定められた表記であり、種々のプログラミング言語ではそれを扱うためのクラスやパッケージが用意されています (例えば Java 11 であれば [Locale クラス][7])。

よく見られる範囲だと BCP 47 による言語表記はせいぜい「言語名 + 地域名」ぐらいで単純なのですが、実際はもっと複雑な表記が可能であり、それに関連して先日 go-gitea/gitea で [Login with Safari: Error updating user language][1] という issue を見つけました。ここではその内容を抑えながら BCP 47 の内容のうち extension (特に u-extension) という概念について見ていきたいと思います。

### 見つけた issue について

gitea の [Login with Safari: Error updating user language][1] ですがざっくりとした起票内容は「golang.org/x/text/language の Matcher を使用した言語選択を行なうときに、よくわからない suffix を付与された値が返されることがある」というものです。

language パッケージの Matcher は例えばシステムがサポートする言語一覧からユーザの希望に合わせて言語を選択する際に使えるような機能だと思います (自分でこの関数を使用したことは無いのですが API 的に)。以下は使用例です。

```go
package main

import (
	"fmt"
	"golang.org/x/text/language"
)

func main() {
	// システムのサポートする言語一覧という想定
	supported := []language.Tag{language.MustParse("ja-JP"), language.MustParse("en-US")}
	m := language.NewMatcher(supported)

	// en-US 1 Exact
	fmt.Println(m.Match(language.MustParse("en"), language.MustParse("fr")))

	// en-US-u-rg-gbzzzz 1 High
	fmt.Println(m.Match(language.MustParse("en-GB"), language.MustParse("fr")))
}
```

ここでは仮にシステムが `ja-JP`, `en-US` という言語をサポートしているときに、各ユーザの希望に合わせて何が返されるかという想定でコードを書いています。1 番目の例だとユーザは `en` が `fr` を希望しているということで `en-US` が選択されています。`Mather#Match` の返り値は「選択された言語、選択された言語の index (ここでは supported の index)、選択に対する自信 (どれくらい希望に合わせた選択がされているか)」です。

問題は 2 番目でユーザは `en-GB` か `fr` を希望しているのですが返される言語が `en-US-u-rg-gbzzzz` ということで馴染みのない `u-rg-gbzzzz` という suffix が付けられています。はじめは何か使い方がおかしいのかライブラリのバグなのかと思ったのですが実際はどうやら BCP 47 に則った表記であることがわかりました。

### BCP 47

BCP 47 による言語表記を文書内では language tag (あるいは language identifier) と呼んでおり、syntax を見ると 3 つに大別できそうです。

- langtag (normal language tag)
- privateuse (private use tag)
- grandfathered (grandfatherd tag)

ここでは主に使われているであろう langtag についてのみ注目し、その拡張 BNF を引用します。

```
  langtag       = language
                  ["-" script]
                  ["-" region]
                  *("-" variant)
                  *("-" extension)
                  ["-" privateuse]

  language      = 2*3ALPHA            ; shortest ISO 639 code
                  ["-" extlang]       ; sometimes followed by
                                      ; extended language subtags
                / 4ALPHA              ; or reserved for future use
                / 5*8ALPHA            ; or registered language subtag

  extlang       = 3ALPHA              ; selected ISO 639 codes
                  *2("-" 3ALPHA)      ; permanently reserved

  script        = 4ALPHA              ; ISO 15924 code

  region        = 2ALPHA              ; ISO 3166-1 code
                / 3DIGIT              ; UN M.49 code

  variant       = 5*8alphanum         ; registered variants
                / (DIGIT 3alphanum)

  extension     = singleton 1*("-" (2*8alphanum))

                                      ; Single alphanumerics
                                      ; "x" reserved for private use
  singleton     = DIGIT               ; 0 - 9
                / %x41-57             ; A - W
                / %x59-5A             ; Y - Z
                / %x61-77             ; a - w
                / %x79-7A             ; y - z
```

例えば `ja-JP` は language として ja, region として JP を指定した形式です
(ちなみに language の言語名称の略称が定義されているのは ISO 639, region の 国名コードが定義されているのは ISO 3166-1)。

今回の話に関連するのは extension 部分です。2.2.6 Extension Subtags 冒頭からの引用ですが、

> Extensions provide a mechanism for extending language tags for use in
> various applications.  They are intended to identify information that
> is commonly used in association with languages or language tags but
> that is not part of language identification

ということで language tag には extension により言語に関連した情報を組み込むことが許されています。
例えば `ja-JP-t-...` という language tag があればそれは language に日本語、region に日本、そして t-extension により何かを指定しているということになります。ただこの拡張は (x-extension 以外) アプリケーション毎に独自の意味合いで使えるというわけでは無く何かしらの標準を定めた上で各アプリケーションでサポートすることが期待されているようです。

### u-extension

そうした extension のうち Unicode Consortium により管理されている拡張が u-extension です。

Unicode Consortium は活動の一環として [CLDR][5] (Common Locale Data Repository) というロケール固有のデータを集めたデータベースの管理をしており、それに関連した情報を含めるために u-extension を使用しています。

u-extension の表記法については [RFC 6067][4] の 2.1 Summary に記述されています。

```
   o  An 'attribute' is a subtag with a length of three to eight
      characters following the singleton and preceding any 'keyword'
      sequences.  No attributes were defined at the time of this
      document's publication.

   o  A 'keyword' is a sequence of subtags consisting of a 'key' subtag,
      followed by zero or more 'type' subtags (so a 'key' might appear
      alone and not be accompanied by a 'type' subtag).  A 'key' MUST
      NOT appear more than once in a language tag's extension string.
      The order of the 'type' subtags within a 'keyword' is sometimes
      significant to their interpretation.

      A.  A 'key' is a subtag with a length of exactly two characters.
          Each 'key' is followed by zero or more 'type' subtags.

      B.  A 'type' is a subtag with a length of three to eight
          characters following a 'key'.  'Type' subtags are specific to
          a particular 'key' and the order of the 'type' subtags MAY be
          significant to the interpretation of the 'keyword'.
```

例として `de-DE-u-attr-co-phonebk` が挙げられており、

- 言語名は `de` (ドイツ語)
- 国名は `DE` (ドイツ)
- `u` で u-extension が使われていることがわかる
- `attr` が attribute
- `co-phonebk` が keyword
  - `co` が key
  - `phonebk` が type

という解釈ができます。

具体的に u-extension にどのような情報を含めることができるのかについては同団体が定義する仕様の一つ、 UTS (Unicode Technical Standard) #35 の [3.6.1 Key And Type Definitions][6] にまとめられています。わかりやすいものでいうと `ca` key というカレンダー表記を指定するためのものがあり、例えば `ja-JP-u-ca-japanese` によって和暦を指定した表現になるようです (ちなみにここで `ca` はどうやら字数的に attribute ではなく key として扱われているみたいです。u-extension 内の厳密な syntax は [3.2 Unicode Locale Identifier][8] 内の BNF を参照)。

Go の language パッケージで使用されていたのは `rg` key であり、これは「Region Override specifies an alternate region to use for obtaining certain region-specific default values, instead of using the region specified by the unicode\_region\_subtag」と説明されています。この説明だけだとあまりピンとこないのですが、そのあと例として `en-GB-u-rg-uszzzz` が挙げられており、これは「言語は British English だけど地域特有の設定 (e.g. currency, calendar) は US のものを使用」ということを意味する表記になるようです。

それを踏まえて最初に示した Go のサンプルコードを見直すと、

```go
    // システムのサポートする言語一覧という想定
	supported := []language.Tag{language.MustParse("ja-JP"), language.MustParse("en-US")}
	m := language.NewMatcher(supported)

	// en-US-u-rg-gbzzzz 1 High
	fmt.Println(m.Match(language.MustParse("en-GB"), language.MustParse("fr")))
```

というのは「言語は American English だけど地域特有の設定については英国のものを使用」という意味の表記だと解釈できます。恐らくユーザの希望する言語に `en-GB` があるので language パッケージが (気を利かせて?) そのような指定を入れているのだと思います。実際に language パッケージ内のコードを見ると以下の部分でその処理が行われているのがわかります。

```go
	// in golang.org/x/text/language/match.go
	...
	if w.RegionID != tt.RegionID && w.RegionID != 0 {
		if w.RegionID != 0 && tt.RegionID != 0 && tt.RegionID.Contains(w.RegionID) {
			tt.RegionID = w.RegionID
			tt.RemakeString()
		} else if r := w.RegionID.String(); len(r) == 2 {
			// TODO: also filter macro and deprecated.
			tt, _ = tt.SetTypeForKey("rg", strings.ToLower(r)+"zzzz")
		}
	}
	...
```

正直なところ u-extension がどのくらい認知されているのかわかりませんし、多言語対応をするシステムやアプリケーションのうちどのくらいがこれを考慮して設定をしてくれるのかは少し疑問です。

[1]: https://github.com/go-gitea/gitea/issues/14793
[2]: https://github.com/golang/text
[3]: https://www.rfc-editor.org/rfc/bcp/bcp47.txt 
[4]: https://www.ietf.org/rfc/rfc6067.txt
[5]: http://cldr.unicode.org/
[6]: http://unicode.org/reports/tr35/#Key_And_Type_Definitions_
[7]: https://docs.oracle.com/javase/jp/11/docs/api/java.base/java/util/Locale.html
[8]: https://www.unicode.org/reports/tr35/#Unicode_locale_identifier
