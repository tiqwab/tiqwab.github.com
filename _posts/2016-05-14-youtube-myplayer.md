---
layout: post
title:  "自分用の簡易YouTube Playerを作る"
tags: "javascript, react"
---

YouTubeをいわゆる「作業用BGM」として使用することが多々あるが、幾つかの短い動画をリピート再生したいというときに手動で操作するのも、かといってログインしてプレイリストを作ってどうこうするのも面倒だなと感じていた。ということで以下のような最低限の機能を持つ自分用のYouTube Playerを作成してみることにした。

- 好きな動画を自分のプレイリストとして追加、削除可能
- 動画の追加は対象のYouTube URLを入力することで行う
- プレイリスト中の曲をエンドレスリピートする。とりあえず今はシャッフル再生はできなくてもよい

最終的にできたものはこのような感じ。

<img
  src="/images/youtube-myplayer/youtube-myplayer.png"
  title="youtube-myplayer"
  alt="youtube-myplayer"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

フロントエンドのフレームワークとしてReact, プレイリストの保存はLocalStorage, また埋込YouTube Playerの操作や再生動画の情報取得のためにはそれぞれYouTube Player API, YouTube Data APIを使用している。

### 1. 環境構築

新規プロジェクト用ディレクトリを作成し、各種ライブラリをインストールする。環境の構築に関しては以前に[こちら]({% post_url 2016-04-24-restart-learning-js %})で試したのと同様に行った。また最終的な`package.json`は以下のようになった。

package.json

```json
{
  "name": "youtube-myplayer",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "webpack": "webpack -w",
    "lite": "lite-server --verbose --open dist",
    "start": "concurrently \"npm run webpack\" \"npm run lite\""
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "babel-loader": "^6.2.4",
    "babel-preset-es2015": "^6.6.0",
    "babel-preset-react": "^6.5.0",
    "concurrently": "^2.0.0",
    "eslint": "^2.9.0",
    "eslint-config-airbnb": "^8.0.0",
    "eslint-plugin-import": "^1.6.1",
    "eslint-plugin-jsx-a11y": "^1.0.4",
    "eslint-plugin-react": "^5.0.1",
    "lite-server": "^2.2.0",
    "webpack": "^1.13.0"
  },
  "dependencies": {
    "babel-polyfill": "^6.8.0",
    "react": "^15.0.2",
    "react-dom": "^15.0.2",
    "youtube-player": "^3.0.4"
  }
}
```

#### YouTubePlayer API

埋込YouTubeを操作するAPIとしては[iframe組み込みのYouTube Player API][1]を使用した。ただし、今回は素のAPIをそのまま使うのではなく、そのラッパーライブラリである[youtube-player][2]を使用してみた。使用例や使用感は後述。

```bash
$ npm install youtube-player --save
```

### 2. youtube-playerによるYouTube表示、再生

まずはYouTubeの表示、再生に必要なminimumなコードを確認しておく。youtube-playerライブラリを使用しているので、[iframe組み込みのYouTube Player API - はじめに][3]と同様のことを行わせるためのコードを書いてみた。

index.html

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>YOUTUBE PLAYER</title>
  </head>
  <body>
    <div id="header">
      <h1>YOUTUBE PLAYER</h1>
    </div>

    <div id="player"></div>

    <script type="text/javascript" src="bundle.js"></script>
  </body>
</html>
```

app.js

```javascript
import youtubePlayer from 'youtube-player';

// Cannot access YT.PlayerState from youtube-player (?), so declare constant here.
const PlayerState = {
  ENDED: 0,
  PLAYING: 1,
  PAUSED: 2,
  BUFFERING: 3,
  CUED: 5,
};

const player = youtubePlayer('player', {
  height: '390',
  width: '640',
  videoId: 'M7lc1UVf-VE',
});

player.playVideo();

let done = false;
player.on('stateChange', (event) => {
  if (event.data === PlayerState.PLAYING && !done) {
    window.setTimeout(() => {
      player.stopVideo();
    }, 6000);
    done = true;
  }
});
```

### 3. YouTube Data API v3による動画情報の取得

YouTubeのURLを元に動画情報を取得するためにYouTube Data APIを使用する。[YouTubeの動画情報をData API v3を使って取得する Qiita][4]を参考。

### 4. Reactによる実装

ReactでViewを作成する。大まかな内容としてはTodoアプリ + αという程度なのでそこまで難しい内容は無い。  
ただし、埋込YouTube PlayerとReactの共存方法だけ怪しい。

- `props`で他のcomponentとのデータの受け渡し、更新を行いたい。
  - YouTube PlayerもReactのcomponentとして扱いたいということになる。
- ただしPlayerの描写に関してはYouTube API側で行い、かつ時系列に対し不変なのでReactで対応する必要は無い。

ということを考えて以下のような実装にしたが、これがReact的に正しい方法なのかは自信が無い。

DetailPlayer.js

```javascript
import React from 'react';
import youtubePlayer from 'youtube-player';

// Cannot access YT.PlayerState from youtube-player (?), so declare same constant here.
const PlayerState = {
  NOTSTART: -1,
  ENDED: 0,
  PLAYING: 1,
  PAUSED: 2,
  BUFFERING: 3,
  CUED: 5,
};

class DetailPlayer extends React.Component {
  constructor(props) {
    super(props);
    this.prepared = false;
    this.player = null;
  }

  // Invoked before the initial rendering.
  componentWillMount() {
    const playerTag = document.getElementById('detail-player');

    // Player height and width come from data attributes of playerTag
    this.player = youtubePlayer('detail-player', {
      height: playerTag.dataset.height,
      width: playerTag.dataset.width,
    });

    this.player.on('stateChange', (event) => {
      const playlist = this.props.playlist;
      const playing = this.props.playing;
      const playingIndex = playlist.findIndex((x) => x.id === playing);

      // Play a next video automatically when the previous video ended.
      if (event.data === PlayerState.ENDED) {
        // All methods of youtube-player return Promise.
        this.player.getVolume()
              .then(volume => {
                if (volume !== playlist[playingIndex].volume) {
                  playlist[playingIndex].volume = volume;
                }
                return Promise.resolve(this.player);
              })
              .then(() => {
                const playIndex = playlist.findIndex((x) => x.id === playing);
                const nextPlayingMusic = playlist[(playIndex + 1) % playlist.length];
                this.props.onNextMusic(nextPlayingMusic.id, playlist, true);
              });
      // When player started after the first video is added to empty playlist.
      } else if (event.data === PlayerState.NOTSTART) {
        if (!this.prepared && playlist.length > 0) {
          this.player.cueVideoById(playlist[0].videoId);
          this.player.setVolume(playlist[0].volume);
          this.prepared = true;
        }
      }
    });

    // Cue the first video of playlist when the page is loaded.
    if (!this.prepared && this.props.playlist.length > 0) {
      this.player.cueVideoById(this.props.playlist[0].videoId);
      this.player.setVolume(this.props.playlist[0].volume);
      this.prepared = true;
    }
  }

  // Invoked when a component is receiving new props.
  // Note that coming 'new props' does not mean prop is changed.
  componentWillReceiveProps(nextProps) {
    const playlist = this.props.playlist;
    const playing = this.props.playing;
    const playingIndex = playlist.findIndex((x) => x.id === playing);

    if (nextProps.playing !== this.props.playing) {
      this.player.getVolume()
            .then(volume => {
              if (volume !== playlist[playingIndex].volume) {
                playlist[playingIndex].volume = volume;
              }
              this.props.onNextMusic(playing, playlist, false);
            })
            .then(() => {
              const nextPlayIndex = playlist.findIndex((x) => x.id === nextProps.playing);
              this.player.loadVideoById(playlist[nextPlayIndex].videoId);
              this.player.setVolume(playlist[nextPlayIndex].volume);
            });
    }
  }

  // Return always false not to invoke render method.
  shouldComponentUpdate() {
    return false;
  }

  render() {
    return <div id="detail-player" />;
  }
}

DetailPlayer.propTypes = {
  playlist: React.PropTypes.array,
  playing: React.PropTypes.number,
  onNextMusic: React.PropTypes.func,
};

export default DetailPlayer;
```

### おわりに

- ReactはTodoアプリを作成して喜ぶ程度の習熟度でしかなかったが、そういう意味では今回のYouTube Playerを作成するというのはいい感じにステップアップした難易度に感じた。
- 埋め込みのYouTube Playerだと広告再生が挟まらない？ので、YouTubeのサイトで直接見るより快適。

### 参考

- [Component Specs and Lifecycle - React][5]

### サンプル用リポジトリ

- [youtube-myplayer](https://github.com/tiqwab/youtube-myplayer)

[1]: https://developers.google.com/youtube/iframe_api_reference?hl=ja
[2]: https://www.npmjs.com/package/youtube-player
[3]: https://developers.google.com/youtube/iframe_api_reference?hl=ja#Getting_Started
[4]: http://qiita.com/moshisora/items/4ea23d5abd7b4d852955
[5]: https://facebook.github.io/react/docs/component-specs.html
