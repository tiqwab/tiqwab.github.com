---
layout: post
title: td-agent (fluentd) 設定、監視メモ
tags: "td-agent, fluentd, memo"
comments: true
---

td-agent (fluentd) を導入する上での個人的なメモ

### td-agent 設定まわり

- 全般
  - ntpd のインストール
  - Check ulimit
  - 参考: [Before Installing][6]
- イベント送信側
  - output forward
    - buffer 設定
    - secondary output
  - S3 へのバックアップ
    - buffer 設定
    - secondary output
    - 参考: [Fluentd でアプリログを S3 へ][5]
- イベント受信側
  - bigquery に送信する部分の buffer 設定
  - secondary output

### td-agent 監視まわり

- `td-agent`
  - 監視 (zabbix でやりたい)
    - CPU/Memory
      - td-agent に限らない話なので何とでもなるはず
    - process
      - `proc.num[ruby,,,td-agent]` が 2 でなかったらエラー
      - 参考: [Zabbix 3.2 で td-agent 用のテンプレートを作成する][2]
    - port
      - td-agent に限らない話なので何とでもなるはず
      - 対象 port が LISTEN であることがわかれば良いなら `net.tcp.listen`
      - 参考: [サービス監視について][4]
    - buffer
- `td-agent.log`
  - 監視
    - 自分自身のログを match させてどこかに送ることができるのでそれを利用するのが良さそう
    - 参考: [Logging of Fluentd][7]
  - ローテーション
    - [td-agent.logrotate][3] で設定されていそう
    - 30 日間で rotate している感じか

### fluent-logger

- 現在 [fluent-logger-scala][8] を使用しているが依存ライブラリやロガーが冗長になる部分があり少し微妙なことに
- [fluency][9] を検討したい

### その他

- `sigdump` (JVM でいう SIGQUIT) によるスレッドダンプ取得ができる

参考

- [Monitoring Fluentd][1]
- [Zabbix 3.2 で td-agent 用のテンプレートを作成する][2]

[1]: https://docs.fluentd.org/v0.14/articles/monitoring
[2]: http://qiita.com/wapa5pow/items/d5290979cfa8f6b6a0b6
[3]: https://github.com/treasure-data/td-agent/blob/master/td-agent.logrotate
[4]: http://www.zabbix.jp/node/2451
[5]: http://qiita.com/digitalpeak/items/60da0f8903b7e869b8fe
[6]: https://docs.fluentd.org/v0.14/articles/before-install
[7]: https://docs.fluentd.org/v0.14/articles/logging
[8]: https://github.com/fluent/fluent-logger-scala
[9]: https://github.com/komamitsu/fluency
