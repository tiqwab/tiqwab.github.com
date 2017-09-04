---
layout: post
title: xmonad ことはじめ
tags: "xmonad, haskell"
comments: true
---

前から興味のあった [xmonad][8] におもむろに手を出したので、その記録です。

現状の xmonad の設定ファイルは以下の場所に push しています。

- [xmonad.hs][10]
- [.xmobarrc][9]

環境

- OS: Arch Linux
- Display Manager: LightDM 1.22.0

### インストール

以下の 2 つのパッケージをインストールします。
`xmonad-contrib` は `xmonad` の動作自体には必須では無さそうですが、 xmonad の設定を変更していったりする上で実質無くてはならない存在なので同時にインストールしておきます。

```
$ yaourt -S xmonad xmonad-contrib
```

インストール後 `/usr/share/xsessions/` 以下に xmonad 用のデスクトップエントリファイルができ、LightDM ではログイン時に右上のボタンから xmonad を選択できるようになりました ([参考: ArchWiki ディスプレイマネージャ][11])。

### xmonad の操作

一通りの操作は以下のガイドから学ぶことができます。

- [xmonad: a guided tour][1]

### カスタマイズ

xmonad は初期状態だとまだまだ使いにくい部分も多くカスタマイズを加えたい点が出てきます。
ここではさくっと設定ができた以下のものについて何をやったかまとめておきます。

#### dmenu

`dmenu` をインストールすると `<mod-p>` でアプリケーションの検索、起動が行えるようになります (`mod` はデフォルトでは `<Alt>` キーを表す)。

```
$ yaourt -S dmenu
```

#### xmobar

`xmobar` をインストールするとよくある感じにディスプレイ上部にステータスバーが表示できます。
ここに例えば現在アクティブな画面の情報や CPU, Memory といった情報を出すことができます。

```
$ yaourt -S xmobar
```

xmobar を有効にするためには `~/.xmonad/xmonad.hs` に `main = xmonad =<< xmobar defaultConfig {...}` という記述を足します。

```haskell
import XMonad
import XMonad.Hooks.DynamicLog

main = xmonad =<< xmobar defaultConfig
    { terminal = "urxvt"
    , borderWidth = 3
    }
```

xmobar 以外にも多少記述がありますが、このように xmonad の設定はこの `xmonad.hs` に集約されます。

xmobar 自体の設定はデフォルトでは `~/.xmobarrc` のようです。

設定を変更後は `xmonad --recompile` からの `<mod-q>` で設定を現在動作中の xmonad に反映することができます (設定によってはだめなときもある気がしますが)。

#### gvim

自分は gvim を使うことがあるのですが、xmonad で使用した場合、下部に不自然なスペースができてしまいます。これを直すために [こちら][7] を参考に `.vimrc` に以下の設定を足しました。

```
set guiheadroom=0
```

#### 画面の輝度と音量調節

画面の輝度と音量をキーで調整できるように、ショートカットを作成します。

```haskell
import           XMonad
import           XMonad.Hooks.DynamicLog
import           XMonad.Util.EZConfig    (additionalKeysP)
import           XMonad.Util.Run         (spawnPipe)

main = xmonad =<< xmobar (def
    { terminal = "urxvt"
    , modMask = mod4Mask -- Change mod key to <Super-L> (windows key)
    , borderWidth = 3
    }
    `additionalKeysP` keysP' -- additionalKeysP :: XConfig l -> [(String, X ())] -> XConfig l
    )

-- shortcut key settings
keysP' = [ ("<XF86AudioRaiseVolume>", spawn "pactl set-sink-volume 0 +5%") -- Volume up
         , ("<XF86AudioLowerVolume>", spawn "pactl set-sink-volume 0 -5%") -- Volume down
         , ("<XF86AudioMute>", spawn "pactl set-sink-mute 0 toggle") -- Toggle volume mute
         , ("<XF86MonBrightnessUp>", spawn "xbacklight + 5 -time 100 -steps 1") -- Monitor brightness up
         , ("<XF86MonBrightnessDown>", spawn "xbacklight - 5 -time 100 -steps 1") -- Monitor brightness down
         ]
```

`keysP'` がショートカットキーの定義で、各キーを押した場合に対応するコマンドが実行されるようにしています。 キーの名称は `xev` で確認しました。

実際にショートカットキーを反映させるには `xmonad-contrib` が提供する `additionalKeysP` 関数を使用しています。

### TODO

何となく xmonad を使い始めるのは色々設定が必要で大変そうだというイメージがあったのですが、カスタマイズにものすごいこだわりがあるというわけでなければ思ったよりは敷居が低そうに感じました。

とはいえまだ本格的に移行できる状態ではなく、少なくとも以下の設定はできるように今後ちょくちょく手を入れたいと思います。

- Screen Lock
- 日本語入力
- Display network
- Display battery in xmobar
- Touch pad

### 参考

- [ArchWiki xmonad][6]
- [http://futurismo.biz/archives/2165][2]
- `xmonad.hs` example
  - [http://qiita.com/ssh0/items/b92d9ab61eb5c92faa3e][3]
  - [https://github.com/Hinidu/dotfiles/blob/master/xmonad/xmonad.hs][4]
- `.xmobarrc` example
  - [http://beginners-guide-to-xmonad.readthedocs.io/configure_xmobar.html][5]

[1]: http://xmonad.org/tour.html
[2]: http://futurismo.biz/archives/2165
[3]: http://qiita.com/ssh0/items/b92d9ab61eb5c92faa3e
[4]: https://github.com/Hinidu/dotfiles/blob/master/xmonad/xmonad.hs
[5]: http://beginners-guide-to-xmonad.readthedocs.io/configure_xmobar.html
[6]: https://wiki.archlinuxjp.org/index.php/Xmonad
[7]: https://wiki.archlinuxjp.org/index.php/Vim#gVim_.E3.82.A6.E3.82.A3.E3.83.B3.E3.83.89.E3.82.A6.E3.81.AE.E5.BA.95.E9.83.A8.E3.81.AE.E7.A9.BA.E3.81.8D.E3.82.B9.E3.83.9A.E3.83.BC.E3.82.B9
[8]: http://xmonad.org/
[9]: https://github.com/tiqwab/dotfiles/blob/master/.xmobarrc
[10]: https://github.com/tiqwab/dotfiles/blob/master/.xmonad/xmonad.hs
[11]: https://wiki.archlinuxjp.org/index.php/%E3%83%87%E3%82%A3%E3%82%B9%E3%83%97%E3%83%AC%E3%82%A4%E3%83%9E%E3%83%8D%E3%83%BC%E3%82%B8%E3%83%A3
