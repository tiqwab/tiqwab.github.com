---
layout: post
title:  Vimとパッケージ管理を理解する
tags: "vim, neobundle"
comments: true
---

Vim様には以前からシステム設定ファイルの編集やちょっとした作業でお世話になっており、キー操作や`.vimrc`に書くような基本的なオプションは理解していたのですが、Vimが裏側でどう動いているかをあまり理解せずに使い続けていました。ですが最近Vimの環境を新しく作り直した際、パッケージの管理に関してどこまでがVim標準でできて、どこからがパッケージマネージャのおかげで動いているのかがわからずもやもやとした気分を味わったので、そのあたりを中心にしてVimが裏側でやってくれていることをまとめてみようと思います。

### 環境

- OS: Arch Linux (linux kernel: 4.8.13)
- Vim: 8.0 (2016 Sep 12, compiled Jan  7 2017 09:31:58). Included patches: 1-149
- Vimのパッケージ管理にNeoBundleを使用

### 1 Vim標準の動きの理解

Vimによるプラグイン管理を理解しようと色々調べたところ、ポイントは「runtimepath」と「vimefiles (Vimの設定ファイル達)」のように思いました。ここではまずその2つについて簡単にまとめてみることにします。

#### 1.1 runtimepathオプション

Vimには`runtime <filename>`というコマンドがあり、指定したファイルを読み込み、Vimスクリプトとして実行させることができます。試しにカレントディレクトリに`echo 'foo'`を書いた`foo.vim`を置きそれをruntimeコマンドで読み込んで...みても恐らく何も表示されないと思います。`:help runtime`でヘルプを読むとruntimeで指定するファイル名に関しては以下のような記述があります。

> Read Ex commands from {file} in each directory given by 'runtimepath' and/or 'packpath'.  There is no error for non-existing files.

つまりruntimeコマンドで指定するファイルは'runtimepath'上から読むよということを言っているようです。

runtimepathというのは上記のようにVimが外部のファイルを読み込んだり処理を行う場合に基準とするディレクトリパスの一覧であり、OSでいえば環境変数'PATH'に当てはまるものだと考えればよいと思います。runtimepathの値は`&rtp` or `&runtimepath`に保存されており、`echo &rtp`等で現在の値を確認することができます。

```
:echo &rtp
~/.vim,/usr/share/vim/vim80,/usr/share/vim/vimfiles/after,~/.vim/after
```

標準では上記のように`$VIMRUNTIME`(標準のVim関連ファイルの配置先を表す環境変数)やホームディレクトリの`.vim`下が含まれているのではないかと思います。先程のファイルは`~/.vim`のようなruntimepath下に配置し直せば実行されることが確認できるはずです。

#### 1.2 vimfiles - Vim設定ファイル達

私の場合、現在の`~/.vim`の構造(一部抜粋)は以下のような感じになっています。

```
.vim
  +- ftdetect
  +- ftplugin
  +- plugin
  +- autoload
```

これは私が理解して(そしてときには理解しないまま)Vimの設定を変更してきた結果の構造なのですが、基本的には多くの環境でも同様の構造になっているのではないでしょうか。
`~/.vim`には上述の通りruntimepathが通っているため、これらのディレクトリはVimから見ることのできる設定や機能の集まりであり、それぞれ役目を持っています。以下ではその役目、またそれらのファイルをいつVimが使用するかをまとめてみたいと思います (ここでは上の4つのみに触れますが、各種vimfilesは`:h vimfiles`から確認することができます)。

#### ftdetect

Vimにはfiletypeという概念があり、現在開いているファイルがmarkdown文書なのかpythonスクリプトなのかといったことをこのfiletypeを設定することで判断するようになっています。現在のfiletypeを確認(`:set filetype`)するとわかるように、Vimはこちらが特に設定をしなくても日常でよく使用するファイルに対してはデフォルトでfiletypeを設定してくれるようになっています。もし自分の使用したいファイルに対して設定されない場合、そうした設定はこのftdetectディレクトリにvimスクリプトを配置することで設定することができます。

以上がftdetectの概要であり日常の使用ではほぼこの理解で困ることはないのですが、一つ以前から気になることがありました。

私の`.vimrc`ファイルではNeoBundleによるパッケージ管理の記述のあとに`filetype plugin indent on`というfiletypeコマンドを実行していますが、昔このfiletypeコマンドをNeoBundleより前に実行し、うまく設定が反映されず悩んだことがあります。そのときには`filetype on`によりVimがこれから開こうとするファイルの拡張子を見てfiletypeを判断するということはわかっていたのですが、この処理の詳細は知りませんでしたし、なぜNeoBundleの処理後に呼ばないといけないかまでは理解していませんでした。

というわけで今回そこを探るためにgithubにホストされているVimのソースを見てみました。初めて見るソースなので時間がかかるかもと思っていたのですが、READMEが丁寧なおかげで割と簡単に該当箇所が見つかり、それによるとどうやら`filetype on`コマンド実行時のメインの処理は`$VIMRUNTIME/filetype.vim`を読込み、実行することのようです。

ローカルに存在する`$VIMRUNTIME/filetype.vim`の内容をさらってみると以下のようになっていることがわかります。

```vim
" 色々前処理
" ...
" 省略
" ...

" filetypeの設定開始
augroup filetypedetect

" ...
" 省略
" ...

" 各種filetypeに対するautocmd設定
au BufNewFile,BufRead *.aap setf aap
au BufNewFile,BufRead */etc/a2ps.cfg,*/etc/a2ps/*.cfg,a2psrc,.a2psrc setf a2ps

"...
" 省略
"...

" runtimepath上のftdetect読込
runtime! ftdetect/*.vim

" filetypeの設定終了
augroup END

" 色々後処理
" ...
" 省略
" ...
```

前処理後処理でちょこちょこやっていることはありますが、ファイル全体としてみれば各種filetypeを設定するためのautocmdをひたすら実行しているという内容になっています。標準の状態でもfiletypeがうまく設定されていることが多いのは恐らくこのファイルのおかげなんですね。

このfiletype.vimの末尾には`runtime! ftdetect/*.vim`という記述があるため、`~/.vim/ftdetect`内に作成したスクリプトはここで呼び出され、それは標準の設定より優先されるということがわかります。先の疑問の回答に結びつくのは恐らくこのコマンドで、NeoBundleがパッケージのftdetectを読み込めるようにしてくれる前にfiletypeコマンドを実行してしまったからということになるかと思います。NeoBundleとruntimepathに関する話を後述しているので、そちらを読んでからの方がわかりやすいかもしれません。

#### ftplugin

上でfiletypeについて触れましたが、ftpluginはそのfiletypeに応じた設定を行うための仕組みです。Vimが新規にファイルを開くと、そのfiletypeを設定し、それに応じたftpluginを実行します。Vimが標準で用意しているftpluginは`$VIMRUNTIME/ftplugin`に保存されており、例えば`haskell.vim`は以下のような内容になっています。

```vim
if exists("b:did_ftplugin")
  finish
endif
let b:did_ftplugin = 1

let s:cpo_save = &cpo
set cpo&vim

let b:undo_ftplugin = "setl com< cms< fo<"

setlocal comments=s1fl:{-,mb:-,ex:-},:-- commentstring=--\ %s
setlocal formatoptions-=t formatoptions+=croql

let &cpo = s:cpo_save
unlet s:cpo_save
```

よくある使い方としてはindentやomni補完の設定といったのがあると思いますが、その他にもfiletypeに応じて行いたい処理は何でも書くことができます。

ftpluginに関して少し気をつけないといけないかなと思うのは以下の2点です。

- 新規ファイルがバッファに読み込まれるたびに該当するftpluginが実行されるので、べき等ではない処理を行う場合注意が必要。また基本的にここで実行する処理はバッファを対象としたものにするべき (`setlocal`コマンドやバッファローカルな変数宣言。`:h filetype-plugin`が参考になる)。
- あとからVimがデフォルトで持つftplugin(`$VIMRUNTIME/ftplguin/*.vim`)が読み込まれるため、もし重複する設定がある場合は注意が必要 (`:h ftplugin-overrule`)

ちなみにここまでプラグインやパッケージといった単語を簡単に使っていましたが、Vimのマニュアルではpluginという単語はftpluginディレクトリに置かれるvimスクリプト(filetype-plugin)や後述のpluginディレクトリに配置したvimスクリプト(global-plugin)を指しているようです。

ここではそれに従い、NeoBundleのような機能はパッケージマネージャと呼ぶことにし、プラグインという単語は原則Vimが定義している通りのものを指すことにしています。

#### plugin

上のftpluginが新規ファイルを開くたびに呼ばれていたのに対し、こちらはVim起動時のみに実行されるvimスクリプトの配置場所となります。

個人で使用する場合はここに書くようなものは.vimrcに書いてしまうような気がしますが、パッケージとしては欠かせないディレクトリになります。

#### autoload

これは今回Vimを調べて初めて知った機能だったのですが、Vimは標準機能としてautoloadを使用したvimスクリプトの遅延読込を行うことができます (`:h autoload`)。

autoloadに配置したスクリプトはpluginのようにVimの起動時に読み込まれたり、filetypeに応じて実行されたりはしません。

しかしVimの起動後、例えば`:call foo#bar#func()`のように`#`区切りで関数をcallすると、runtimepath下の`autoload/foo/bar.vim`が(まだ読み込まれていなければ)読み込まれ、その中で定義した`foo#bar#func()`が呼ばれるといった動きになります。

パッケージとしてこの機能を使用すると、例えば以下のようなコマンド定義をpluginに書いておき、実際のパッケージ本体の機能はautoload側で定義しておくことで、実際にパッケージが必要になったときに全体を読み込むような動きにすることができます。

```
command! -nargs=0 CmdFoo call foo#bar#func()
```

通常autoloadで使用するのは関数になると思いますが、実際には変数を参照した場合も同様のことが可能です。一方で変数の定義はautoloadを気にせず行うことができます。一般にパッケージを使用する際には、.vimrcで`g:foo#config_bar = 1`のようにパッケージに関する設定を行い、その後でパッケージが提供するコマンドを実行することで柔軟なパッケージの設定と遅延読込を実現していることが多いと思います。

### 2 Vimのパッケージマネージャに求められるもの

これまでの内容でVimがruntimepath下の指定のディレクトリ、ファイルを読み込むことで自身の設定、動作を決めているということがわかりました。

パッケージとして新しい機能を足す際もこの仕組みはそのまま使用したいと思うのが自然だと思います。実際NeoBundle等でインストールするパッケージの構造を見るとpluginやautoload等vimfilesで構成されています。

そうなるとあと必要なのはそうしたパッケージのルートディレクトリをruntimepathに含めることで、これがいってしまえばパッケージマネージャに最も求められる機能ということになるのだと思います。

NeoBundleに関する話をすれば、通常パッケージの管理として.vimrc内で`NeoBundle`コマンドや`NeoBundleLazy`コマンドを使用するかと思いますが、ここで追加されたものが適切なタイミングでruntimepathに含まれるようになっています。

実際にNeoBundleとruntimepathの挙動を見てみるために、以下のvimスクリプト達を用意しました。

```
<home directory>
  +- .vimrc
  +- .vim
      +- plugin
          +- checkrtp.vim
      +- ftplugin
          +- foo.vim
      +- bundle
          +- foo-plugin
              +- plugin
                  +- echo.vim
              +- ftplugin
                  +- foo.vim
              +- ftdetect
                  +- foo.vim
```

`.vimrc`ではNeoBundleに'foo-plugin'という(ほぼ何もしない)パッケージを用意し、これを管理してもらうようにしました。なおここでは簡易的にNeoBundleではなくNeoBundleLocalを利用しています。またruntimepath挙動を確認するために`neobundle#end()`を呼ぶ前後でその時点のruntimepathを保存しています (`g:rtp_before_end`, `g:rtp_after_end`)。

```vim
" NeoBundle
let s:noplugin = 0
let s:bundle_root = expand('~/.vim/bundle')
let s:neobundle_root = s:bundle_root . '/neobundle.vim'
if !isdirectory(s:neobundle_root) || v:version < 702
    let s:noplugin = 1
else
    if has('vim_starting')
        execute "set runtimepath+=" . s:neobundle_root
    endif

    call neobundle#begin(s:bundle_root)
    NeoBundleFetch 'Shougo/neobundle.vim'

    " Add foo-plugin
    call neobundle#local('~/.vim/bundle', {}, ['foo-plugin'])

    NeoBundleCheck

    " Store runtimepath before neobundle#end
    let g:rtp_before_end = &rtp
    call neobundle#end()
    " Store runtimepath after neobundle#end
    let g:rtp_after_end = &rtp
endif

filetype plugin indent on
```

`.vim/plugin/checkrtp.vim`では上で保存したruntimepathを確認するようにします。

```vim
echo 'echo from ~/.vim/plugin/checkrtp.vim'
echo '  rtp_before_end is ' . g:rtp_before_end
echo '  rtp_after_end is ' . g:rtp_after_end
```

`.vim/ftplugin/foo.vim`はfoo-pluginで設定するfiletype用のftpluginです。

```vim
echo 'echo from ~/.vim/ftplugin/foo.vim'
```

`.vim/bundle/foo-plugin/plugin/echo.vim`はパッケージのpluginが読み込まれることを確認するために配置しています。

```vim
echo 'echo from ~/.vim/bundle/foo-plugin/plugin/echo.vim' 
```

`.vim/bundle/foo-plugin/ftplugin/foo.vim`はパッケージのftplugin確認用です。

```vim
echo 'echo from ~/.vim/bundle/foo-plugin/ftplugin/foo.vim'
```

`.vim/bundle/foo-plugin/ftdetect/foo.vim`ではfooというfiletypeを新規に設定しています。

```vim
au BufRead,BufNewFile *.foo set filetype=foo
```

この状態で`vim hoge.md`によりVimを起動すると以下のようなecho文が確認されます (一部編集)。

```
echo from ~/.vim/plugin/checkrtp.vim
  rtp_before_end is ~/.vim,~/.vim/bundle/.neobundle,/usr/share/vim/vimfiles,/usr/share/vim/vim80,/usr/share/vim/vimfiles/after,~/.vim/after,~/.vim/bundle/neobundle.vim
  rtp_after_end is ~/.vim,~/.vim/bundle/foo-plugin/,~/.vim/bundle/.neobundle,/usr/share/vim/vimfiles,/usr/share/vim/vim80,/usr/share/vim/vimfiles/after,~/.vim/after,~/.vim/bundle/neobundle.vim
echo from ~/.vim/bundle/foo-plugin/plugin/echo.vim
```

この表示から、

- `neobundle#end()`によりruntimepathへのパッケージの追加が行われること
- パッケージのpluginもVimから認識されていること
  - pluginの実行は.vimrc読込終了後に行われる

がわかります。

またこの状態から`hoge.foo`を開くと以下のようにftpluginで設定したecho文が表示され、パッケージのftdetect, ftpluginの設定が正しく反映されていることが確認できます。

```
echo from ~/.vim/ftplugin/foo.vim
echo from ~/.vim/bundle/foo-plugin/ftplugin/foo.vim
```

このようにパッケージマネージャはVimが持つruntimepathを利用することで様々なプラグインやパッケージの追加を簡単に行えるようにしているといえます。

### Summary

- Vimはruntimepath上のvimfilesを実行することで各種設定や機能の追加を行っている
- (NeoBundleのような)パッケージマネージャはパッケージという単位でまとめられたvimfilesをruntimepathへ追加することでVimから認識できるようにしている

### 参考

- Vimマニュアル全般
  - vim script (`:h usr_41.txt`)
  - write-plugin (`:h write-plugin`)
- [Vimスクリプトを書いてみよう](https://www.kaoriya.net/blog/2012/02/19/)
- [Learn Vimscript the Hard Way](http://learnvimscriptthehardway.stevelosh.com/)
