---
layout: post
title: Scrapbox 用のブラウザアドオンを作成してみた
tags: "webextension,scrapbox"
comments: true
---

自分は作業中の記録だったり読書メモだったりを [Scrapbox][20] で記録しているのですが、ドキュメントで手軽に図も使いたいと思い  Diagrams.net による図作成と Scrapbox を連携した [scrapbox-diagramsnet-extension][21] というブラウザアドオンを作成してみました。

以下は初めて Scrapbox に用意されている拡張機能であったりブラウザアドオンの開発に触れてみて感じたことを整理した内容になります。

- [作成したアドオンについて](#about-created-addon)
- [Diagrams.net の外部連携](#about-diagramsnet)
- [Scrapbox 機能拡張方法の選択](#scrapbox-extension)
- [ブラウザアドオン作成について](#create-browser-addon)
- [本アドオン作成時の困難](#trouble-create-addon)

---

<div id="about-created-addon" />

### 作成したアドオンについて

作成したアドオンが持つ機能は単純で以下の 2 つのみです。

- Scrapbox ドキュメント編集中に新しく Diagrams.net で図を作成できる
- 作成した図をクリックすることで再編集できる

利用デモとして [scrapbox-diagramsnet-extension][21] の README に動画も載せてあります。

アドオンでは外部サービスとして Diagrams.net と Gyazo を利用します。Diagrams.net で図作成、編集を行い、出力した画像を Gyazo にアップロード、Scrapbox ドキュメント上ではその URL をリンクすることで画像の表示を行います。

<figure>
  <img
    src="/images/create-scrapbox-extension/addon-abstract.png"
    title="addon-abstract"
    alt="addon-abstract"
    style="display: block; margin: 0 auto; border: 1px solid #eee; width: 400px; height: 400px; object-fit: none;"
  />
</figure>

<div id="about-diagramsnet" />

### Diagrams.net の外部連携

[Diagrams.net][1] (旧 Draw.io) は Web ブラウザ上で多彩な図を書くための Web アプリケーションであり、例えば各種 UML 図やインフラ、ネットワーク構成図を書くために利用できます。図を作成するときに利用するツールは色々ありますが、ログイン不要で画像としてデータを出力したいという場面には適したツールの一つになると思います。

今回作成したアドオンでは Diagrams.net の提供する Embed mode というのを利用して、図の作成や画像データの出力を行いました。

#### Embed mode

Diagrams.net のよく知られた利用方法は [https://app.diagrams.net][2] を訪れてその上で図の作成をするものですが、それとは別に [Embed mode][3] と呼ばれる方法があります。これは `https://embed.diagrams.net` をiframe に埋め込んで利用するというものであり、あたかも自身のアプリケーションの一部として図作成機能を提供することができます。 Embed mode については先程のリンク先に簡単な例もありますが、他に Wiki ツールの [Growi][4] で連携機能として利用されていたりもするのでそちらを見るとどのような機能か理解しやすいかもしれません。

ちなみに Diagrams.net には API 機能は無いので Diagrams.net を外部サービスと連携する場合は Embed mode を使うのが普通になると思います (もちろん Diagrams.net は OSS なので自分で改変してホストすることは可能)。

#### 出力ファイル

出力フォーマットに XML があることから推測される通り、Diagrams.net で作成したデータは実際には XML で表現されています。Diagrams.net は一度出力した画像ファイルを再度取り込んで編集できるのですが、これは画像出力時、内部に自身の XML データを埋め込みそれを読むことで実現しているようです。例えば png の場合は [tEXt 領域][22] が利用されます。

Embed mode では iframe で開いた `embed.diagrams.net` との初期化プロセスの中でこの画像データを送りつけることで画像編集機能を実装できます。

[公式実装例][5] からの抜粋:

```javascript
    # 事前に iframe で embed.diagrams.net を読み込んでいる前提
    # content に png データを Base64 エンコードしたものが含まれている
    iframe.contentWindow.postMessage(JSON.stringify(
    {
        action: 'load',
        autosave: 0,
        xmlpng: 'data:image/png;base64,' + content
    }), '*');
```

<div id="scrapbox-extension" />

### Scrapbox 機能拡張方法の選択

Scrapbox に何らかの機能拡張を行う手法として以下の 2 つを検討しました。

- ブラウザアドオンの作成
- Scrapbox が提供する UserScript 機能の利用

ブラウザアドオンはブラウザ自体にインストールして使用する拡張であり、かなり柔軟にやりたいことが実装できます。一方で利用者としては別途インストールが必要であったり、許可される動作を把握していないと怖かったりと利用のハードルは上がります。

[UserScript][6] はユーザ自身で JavaScript コードを記述し Scrapbox の機能を拡張することができます。例えば以下のような機能を実装することができます。

- [クリックでON/OFFできるチェックボックス][7]
- [ScrapboxでMermaidを使う][8]

UserScript では [UserScriptから使えるscrapboxのobject][9] で紹介されているように `scrapbox` オブジェクトを通してページ内容を取得したり、ページ内容の更新をトリガーとするイベントハンドラを登録するといったことができます。

UserScript はお手軽に利用できて良いのですが、今回は iframe を利用したい関係上、UserScript では Scrapbox で設定されている iframe-src CSP に違反してしまうのでブラウザアドオンとして実装することにしました (ブラウザアドオンにはページの CSP は適用されず自身で別にポリシーを管理する。その点でも信頼できないコードを実行したときのリスクは UserScript の方が少ないので可能なら UserScript で実装した方がいいと思う)。

<div id="create-browser-addon" />

### ブラウザアドオン作成について

今回作成したような GUI 設定画面も持たず極単純な機能を目指すと以下の 3 つの要素のみでアドオンは構成されます。それぞれの構成要素の概要と自分が気付いた実装上の注意点をまとめてみました。

- manifest.json
  - アドオンの説明やバージョンといったメタデータを記述する
  - アドオンの要求するパーミッションや従うべき CSP、許容するリクエスト先の指定といったセキュリティ関連の指定もできる
- content script
  - 表示中のページに応じて実行したい処理を記述する
  - manifest.json に対象のページとともにスクリプトファイルを指定する
  - 表示中のページ内 DOM 要素に自由にアクセスが可能
  - 表示中のページで実行されているコードにはアクセスできない
  - 直接アクセスできる WebExtensions API には制限がある
  - 外部サイトへのリクエスト時は表示中のページをオリジンとして CORS が適用される
- background script
  - 表示中のページに関係なく常駐する処理を記述する
  - manifest.json にスクリプトファイルを指定する
  - DOM にアクセスできない
  - すべての WebExtensions API にアクセスできる
  - manifest.json で許可された外部サイトへリクエスト可能

雰囲気を理解する助けにはなると思うので実際に作成したアドオンの manifest.json を以下に記載します。
これは 1 つの background script と content script を含み、content script は scrapbox.io を対象にしています。

```json
{
    "name": "scrapbox-diagramsnet-extension",
    "version": "0.0.2",
    "description": "Web browser extension to integrate Diagrams.net (Draw.io) editor with Scrapbox",
    "homepage_url": "https://github.com/tiqwab/scrapbox-diagramsnet-extension",
    "manifest_version": 2,
    "minimum_chrome_version": "80",
    "browser_specific_settings": {
        "gecko": {
            "id": "{313f6a66-0678-4416-b6df-6714d43c24d9}",
            "strict_min_version": "80.0"
        }
    },
    "permissions": [
        "https://scrapbox.io/*",
        "https://upload.gyazo.com/*",
        "https://i.gyazo.com/*",
        "https://gyazo.com/*"
    ],
    "content_scripts": [
        {
            "matches": ["*://scrapbox.io/*"],
            "js": ["diagrams.js"]
        }
    ],
    "background": {
        "scripts": [
            "background.js"
        ]
    }
}
```

manifest.json がアドオンとしてのメタデータ管理場所、実行したい処理は content script あるいは background script として実装します。2 種類のスクリプトの使い分けは、今回のようにある特定ページの機能を拡張したいという場合は、「基本は content script に記述、ただし必要な WebExtensions API やクロスオリジンなリクエストに応じて background script に処理を委譲」というようになると思います。

WebExtensions API はブラウザの各種情報の取得や操作ができる API で、例えばタブに関する API だと新しくタブを開いたり表示中のタブで動的にスクリプトを実行したりといったことができます。利用する API は manifest.json の `permissions` に記述する必要がありますが、今回は必要なものが無かったので特に指定をしていません。API は WebExtensions の仕様に含まれるため、理想的には同じソースコードで Firefox や Google Chrome に対応したアドオンを実装することができます。

(理想的、という枕詞をつける意味合いとして現状全く気にしなくていいというわけではないので。例えばこの API は Firefox では `browser` オブジェクトを介して、Chrome では `chrome` を介してとズレた名前で提供されていたり、 Firefox 側だけが Promise をサポートしていたりする。これらの違いは [webextension-polyfill][11] であまり気にならなくはなるが)

ブラウザ上で動作するスクリプトというと CORS や CSP がどうなっているかも気になるところだと思いますが、それも manifest.json に関連した設定があります。

CSP に関して言うとアドオンは表示中のページとは別に manifest.json の `content_security_policy` で指定した内容で定義されます。未指定の場合デフォルトでは `script-src 'self'; object-src 'self';` となります。

CORS は現状ややこしい部分があるのですが、実装では「外部サイトへのリクエストは background script からのみとし、アクセス先を `permissions` で明示的に許可する」という方針にするのがわかりやすいと思います。content script と background script 間では非同期メッセージングの仕組みが用意されているので、content script 側でリクエストが必要になったときに background script に処理を委譲するということができます。

例えば作成したアドオンでは「URL から Gyazo にアップロードした画像データを取得したい」という処理を行うために、content script, background script でそれぞれ以下のような処理を実装しました。

content script 側:

```javascript
// background script へメッセージ送信
const result = await browser.runtime.sendMessage({
    type: 'loadImage',
    url,
});

// result.data に画像データが含まれる
```

background script 側:

```javascript
// メッセージハンドラ登録
browser.runtime.onMessage.addListener((message, _sender) => {
    switch (message.type) {
        case 'loadImage': {
            // Fetch API を利用して画像データを取得、Base64 としてエンコードしている
            return fetchGyazoImage(message.url)
                .then(blob => blobToBase64(blob))
                .then(encoded => ({
                    data: encoded,
                }));
        }

        // other cases...
    }
});
```

ややこしい部分についての話をすると、Firefox と Google Chrome では現在サポートする WebExtension 仕様のバージョンが異なり、Firefox は Manifest v2 で Chrome は v3 です。その違いによりアドオンで許可する外部リクエスト先の指定を行う際、Firefox 向けアドオンでは manifest.json の `permissions` を利用するが Chrome 向けでは `host_permissions` を利用する必要があります。また Firefox は v100 までは content script も明示的に許可したホストにアクセスできてしまう (表示中のページをオリジンとした CORS に従わない) ので、Chrome 使用時と挙動が異なるという点も注意です。v101 からは同一の挙動になりますがそのあたりの差異でトラブルに陥らないためにも外部サイトへのリクエストは background script でのみ行うと決めてしまう方がわかりやすいと思います。

また content script では DOM に自由にアクセスすることはできるのですがページ本来のスクリプトへのアクセスはできません。これは今回の場合 UserScript では利用できていた `scrapbox` オブジェクトへのアクセスはアドオンではできないということになります。そのため Scrapbox の UI を触る場合にそれが API として提供されていたとしても直接 DOM をいじるしかありません。

より一般的な「アドオンってどうやって作るの」については [MDN のドキュメント][10] を中心に読むのが一番だと思います。
Google 側にも同様のドキュメントはあるかもしれませんが、基本的な機能や API はブラウザ間で WebExtensions API として仕様化されているので、自分は上のドキュメントだけで Google Chrome でも動作するアドオンを作成することができました (ただサポートする仕様バージョンの違いでビルド周りに少し困っていますが)。

<div id="trouble-create-addon" />

### 本アドオン作成時の困難

最後に今回アドオンを作成して特に悩んだ点や苦労した点についてです。

#### アドオンで編集可能な画像の見分け方

本アドオンで実装したい内容の一つに追加した図を再編集するというものがありました。Scrapbox では元々追加した画像を独自のエディタで編集、Gyazo にアップロードする機能 (下記画像の「Draw」ボタン) があるので、これに倣い Diagrams.net で作成した図の場合のときだけ「Edit on diagrams.net」という編集用のボタンを配置したいと考えました。

<figure>
  <img
    src="/images/create-scrapbox-extension/scrapbox-edit-image-button.png"
    title="scrapbox-edit-image-button"
    alt="scrapbox-edit-image-button"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="50%"
  />
</figure>

「Diagrams.net で編集可能な画像か否か」の判定方法はいくつか考えられますが、ここではひとまず [Scrapbox の文字装飾記法][12] を利用して目印となる CSS class を付与するという形で機械的に判別可能にしました。

Scrapbox では `[* foo]` や `[- bar]` のような記法が存在しますが、これらには CSS 上では `deco-*` や `deco--` のように `deco-` をプレフィックスとした class が割り当てられます。公式に使われるこれらの記号以外にも実は `[| foo]` や `[# bar]` でも同様の形式で CSS class が当たるのでこれによりユーザがドキュメントの装飾を拡張する余地が存在します。この機能は UserCSS と呼ばれ上記のリンク先でも説明されている内容です。ただし注意点として「記号は予告なく増減する」らしいのでもしかすると将来的には使用した記号に異なる意味が公式に付与される可能性等はあります。

とはいえその可能性はあまり高くないと見たこと、簡便に実装できることから本アドオンでは `[| <img_url>]` のようにして、`deco-|` class を持つ画像はアドオンで作成した画像であるという目印として利用しました。

#### SPA 対応

Scrapbox は [React で書かれている][13] ようなのですが、そのような SPA で DOM を操作する content script を実装するのは一筋縄では行かないと感じました。content script の実行タイミングは多少 manifest.json の `content_script.run_at` で指定する余地があるのですが、基本的には「DOM の読み込みが完了したか」程度のタイミング指定しかできず、これは SPA ではあまり意味のあるタイミングにはなりません。

調べてみるとブラウザアドオンの場合は [Chrome 拡張機能を React などで構成された SPA で動作させる方法][14] のように `browser.tabs.onUpdated.addListener` を利用してタブの更新をトリガにしてスクリプトを実行させるというやり方があるようです。ただ自分の場合は実際にやってみるとイベントの発生タイミングが早すぎて Scrapbox 内でページ遷移する場合にうまく動作せず断念しました。

最終的には [MutationObserver][15] で対象とする要素の出現を監視するという愚直な手段を取りました。はじめて使用するので性能面で問題にならないか心配でしたがいまのところ使っていて特に気になることはありません。

#### アドオン開発環境

今回アドオンを作成する際に雛形として [fregante/browser-extension-template][16] を利用しました。これはアドオンを開発する上で必要なライブラリやツール等の開発環境が一通り揃っており便利ではありましたが、ブラウザ間で現在サポートする WebExtensions 仕様のバージョンが異なるという外部的要因のせいで現状少し使い辛さも感じました。

2022-05-24 現在 Firefox と Chrome でサポートする WebExtensions 仕様がそれぞれ Manifest v2, v3 と異なるため、それぞれのブラウザ用に別のビルド設定を求められる、例えば異なる manifest.json を用意する必要があります。ただテンプレートで使用しているビルドツールである Parcel がシンプルさを求めていることやバグの存在 ([parcel-bundler/parcel/issues/8071][19]) により現状ではこのテンプレートでクロスブラウザ対応のアドオンを新規作成するのは難しい状況になっています (バグ自体は直せばいい話ですが、v2, v3 両対応は Parcel 自体が WebExtension ビルドの認識に `manifest.json` という名前を決め打ちしている部分があるのでそれが思想としてそうなっているのだとすると微妙かもしれないなと)。

少なくとも Manifest v3 についてはそのうち (2022 年内予定) 解決する問題だとは思いますが、もしいまから新しくアドオンを開発するならば上記 issue が解決するまではビルドツールには Parcel ではなく webpack なり別のツールを使う方が良いかなと思います。

[1]: https://www.diagrams.net/
[2]: https://app.diagrams.net/
[3]: https://www.diagrams.net/doc/faq/embed-mode
[4]: https://docs.growi.org/en/guide/features/drawio.html
[5]: https://github.com/jgraph/drawio-github
[6]: https://scrapbox.io/help-jp/UserScript
[7]: https://scrapbox.io/customize/%E3%82%AF%E3%83%AA%E3%83%83%E3%82%AF%E3%81%A7ON%2FOFF%E3%81%A7%E3%81%8D%E3%82%8B%E3%83%81%E3%82%A7%E3%83%83%E3%82%AF%E3%83%9C%E3%83%83%E3%82%AF%E3%82%B9
[8]: https://cockscomb.hatenablog.com/entry/2022/02/21/101812
[9]: https://scrapbox.io/scrapboxlab/UserScript%E3%81%8B%E3%82%89%E4%BD%BF%E3%81%88%E3%82%8Bscrapbox%E3%81%AEobject
[10]: https://developer.mozilla.org/ja/docs/Mozilla/Add-ons/WebExtensions
[11]: https://github.com/mozilla/webextension-polyfill
[12]: https://scrapbox.io/help-jp/%E6%96%87%E5%AD%97%E8%A3%85%E9%A3%BE%E8%A8%98%E6%B3%95
[13]: https://scrapbox.io/shokai/Scrapbox%E3%81%AE%E9%96%8B%E7%99%BA_-_React_&_Websocket%E3%81%A7%E4%BD%9C%E3%82%8B%E3%83%AA%E3%82%A2%E3%83%AB%E3%82%BF%E3%82%A4%E3%83%A0Wiki
[14]: https://r17n.page/2019/10/27/chrome-extension-with-spa/
[15]: https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver
[16]: https://github.com/fregante/browser-extension-template
[17]: https://github.com/fregante/browser-extension-template/issues/78
[18]: https://blog.mozilla.org/addons/2022/05/18/manifest-v3-in-firefox-recap-next-steps/
[19]: https://github.com/parcel-bundler/parcel/issues/8071
[20]: https://scrapbox.io/
[21]: https://github.com/tiqwab/scrapbox-diagramsnet-extension
[22]: https://www.w3.org/TR/2003/REC-PNG-20031110/#11tEXt
