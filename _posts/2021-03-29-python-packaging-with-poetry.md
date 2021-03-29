---
layout: post
title: Python パッケージングと Docker イメージの作成
tags: "python"
comments: true
---

何だかわかりやすいタイトルをつけることができなかったのですが、やりたいことは Python で書いたアプリケーションを Docker コンテナとしてデプロイするときにどのようにパッケージングやイメージのビルドを行なうのがいいのかなというものです。

[sample-python-packaging][3] に実際にお試しで用意したプロジェクトを置いていますが、自分が軽く調べた感じだと現在 Python で新しくプロジェクトを作成するなら、

- Poetry をビルドツールとして使用して wheel 形式の bdist を作成する
- 本番用イメージのビルド時にそれを `pip install` する

というのが簡単に自分の要望を満たせるやり方でした。

Python の配布物の形式は全然詳しくなかったのですが [Python パッケージングの標準を知ろう][1] がわかりやすくまとめられていてよかったです。

今回の目的に関連した部分だと、

- Python には sdist と bdist という 2 種類のパッケージ配布形式がある
- bdist がビルド済み配布物で、展開、配置するだけで使用できる (通常 `pip install` 実行時には wheel ファイルを持ってきている)
- sdist はソースコード配布物で、ユーザは sdist から bdist をビルドしてインストールすることができる
- sdist からのビルドには少し前までは `setuptools` と `setup.py` を使用する必要があった
- 最近では `pyproject.toml` というビルド設定を記述するためのファイルが標準で定義され、Poetry のようなビルドツールを使うとそこから簡単に wheel ファイルを作成することができる

あたりの内容を学ばせてもらいました。

既存のアプリケーションやライブラリだとやはり `setuptools` と `setup.py` で管理されているものが多いのでその知識がいらなくなるということは当分無さそうですが、これから新規に作成するアプリケーションでは Poetry を使うと大分ビルドの手間が省けそうだなと感じました。

[1]: https://engineer.recruit-lifestyle.co.jp/techblog/2019-12-25-python-packaging-specs/ 
[2]: https://www.python.org/dev/peps/pep-0427/
[3]: https://github.com/tiqwab/example/tree/master/sample-python-packaging
