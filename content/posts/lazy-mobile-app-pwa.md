---
title: "怠惰に作るモバイル向けアプリ"
date: 2026-11-26
description: 適当にAIに作らせた簡単なアプリをいちいちストア申請するの面倒くさくね？という話
tags: [web, PWA, mobile]
---

## アプリを作りたいがいろんなハードルが高い

Android向けは、まぁ何とかなる。

iPhone向けのアプリは、SwiftやらXcodeやら覚えることが多すぎて、簡単なアプリを作るのだけでも一苦労ある。

### Apple Developer Programが高すぎる

年額$99なので、普通に払えばいいと思うんですが、自分が欲しいだけのアプリを作って公開したいだけにしては高すぎると思うんですよね。

しかもストアに公開しようものなら、プライバシーポリシーのURLを用意し、審査員の気まぐれなリジェクトに怯える日々が始まるわけですよ。

最近のiPhoneなんか、画面のサイズやら折りたたみやらで大変なことになってますしね。

ちなみにiPhoneは7を1台と古いiPadしか持ってないので、本格的にアプリ開発しようと思ったら端末を買うところからはじめないといけないため、これもまた厳しい。
どなたかがiPhone Duoあたりを譲ってくれるのでしたら、それを売って適当なiPhoneを買うので、Duoください。

## PWAの「むしろこれで良くないか」感

そこで、Webアプリをスマホアプリのように見せる技術がある。
何を今更と思った人は、対象読者ではないと思われる。

PWAの詳しい話は、MDNでも見てほしい。

- [プログレッシブウェブアプリ (PWA) | MDN](https://developer.mozilla.org/ja/docs/Web/Progressive_web_apps)

要は適当なホスティングサービスでHTTPSでのWebページ公開ができれば、スマホのホーム画面にアプリとして配置できる技術セットみたいな感じですね。

### PWAとして動かすための最低要件

いつからか分かりませんが、以前より本当に簡単になりました。

1. HTTPS環境
    - セキュリティ上の理由で必須（ローカル開発時のlocalhostだけは例外）
    - GitHub Pages、Cloudflare Pages、Vercelなんかは自動でHTTPSなので問題なし
2. `manifest.json`
    - index.htmlのheadタグ内に`<link rel="manifest" href="...">`を書いてリンクさせる
3. アイコン画像（PNGなど）
    - 最低限512x512の画像を1枚用意して使い回す
4. JavaScriptで、Service Workerの登録を書く（なくても良い）
    - 以前はインストール条件に必須だったはず
    - 現在の主要ブラウザでは`manifest.json`だけで良いらしい
    - ただ、場合によっては裏側でService Workerを要求したり、インストール促進バナーの出やすさに関わったりするため、「中身が空のsw.jsをindex.htmlとリンクしておく」のが一番無難そう

### 必要なファイル

画像は各自で用意してもらう必要があるが、3つのテキストファイルと画像だけでアプリが作れると思ってもらって良いです。

ファイル構成として、以下のような感じが最低要件です。

```text
プロジェクトディレクトリ/
├── index.html
├── manifest.json
├── sw.js
└── icon.png  ← 512x512の画像を1枚だけ用意
```

#### `index.html`

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>雑</title>
  
  <link rel="manifest" href="./manifest.json"><!-- マニフェストを読み込む（重要） -->
  <link rel="apple-touch-icon" href="./icon.png"><!-- iOS用のアイコン設定（念のため） -->
</head>
<body>
  <h1>これは雑なPWAです</h1>
  <p>雑なメッセージを表示するだけのアプリです。</p>
  <button onclick="alert('動いた！')">押してみる</button>

  <script>
    // Service Workerを登録する
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('./sw.js');
      // エラー処理は省略（本当は書いたほうが良さそう）
    }
  </script>
</body>
</html>
```

#### `manifest.json`

```json
{
  "name": "雑",
  "short_name": "雑",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#ffffff",
  "icons": [
    {
      "src": "./icon.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

#### `sw.js`: 中身は何も書いてなくても最低限では動作する

```js
// インストール時の処理
self.addEventListener('install', (event) => {
  console.log('Service Worker: インストール完了'); // とりあえずログ放流
});

// リクエスト時の処理
self.addEventListener('fetch', (event) => {
  // オフライン対応したくなったらここにキャッシュ処理
});
```

### 「〇〇したい」ときの追加要件リスト

基本のPWAを作ったあと、欲が出てきたときに追加する技術要素です。
いくつかは調べただけで未検証だったりしますし、ブラウザごとに使用できる機能が異なりますが、Androidならとくに問題はないはずです。

1. オフライン（圏外・機内モード）でも動かしたい
    - `Cache API`（Service Worker内に記述）
        + 初回アクセス時にHTML・CSS・JS・画像などをブラウザのキャッシュ領域に保存し、通信できないときはそこから返す処理を書く
2. スマホにプッシュ通知を送りたい
    - `Web Push API` ＋ Push通知を送信するサーバー（バックエンド）
        + サーバー側から通知を飛ばす仕組みが必要（Firebase Cloud MessagingやOneSignalなどの外部サービスを使うと楽らしいが未検証）
        + iOSでも16.4以降で対応したが、「ユーザーが一度ホーム画面に追加していること」が必須条件とのこと
3. アプリ内にデータを保存したい（メモ帳・設定値など）
    - `localStorage`または`IndexedDB`が必要
        + サーバー（DB）を用意しなくてもスマホ本体にデータを残せる
        + テキストや小さなJSONなら`localStorage`で十分、画像や大量のデータなら`IndexedDB`が良さそう
4. スマホの共有メニュー（「共有」ボタン）の送り先に自分のアプリを出したい
    - `Web Share Target API`（`manifest.json`に`share_target`を追記する）
        + X（旧Twitter）やブラウザの「共有」から、自分の雑アプリにURLやテキストを直接送り込めるようになる
        + Android Chromeなどで有効、iOSは現時点で未対応
5. カメラ・GPS・マイク・バイブレーション機能を使いたい
    - 追加の仕組みは不要（ブラウザ標準のWeb APIをJSから叩くだけ）
        + `navigator.geolocation`（位置情報）や`navigator.mediaDevices.getUserMedia`（カメラ）など、普通のWebサイトと同じコードで動く
        + ただし、初回利用時にブラウザ標準の「許可しますか？」ポップアップが出るはず
6. アプリ起動時に画面を縦向き・横向きで固定したい
    - `manifest.json`の`orientation`プロパティ（例：`"orientation": "portrait"`）
        + 1行足すだけで画面回転をロックできる
        + ただしiOS Safariでは効かないことがある（Safari滅んでくれねぇかな）
7. アプリアイコンの右上に通知バッジ（赤い数字）を出したい
    - `Badging API`（`navigator.setAppBadge(n)`）
        + JSで数値を渡すだけでホーム画面のアイコンに未読件数のようなバッジをつけられる

## 実際作ったもの

完全に自分用なので、ブラッシュアップもしていない雑さです。

UIのデザインにAI臭がします。

### 電子レンジあたため時間計算ツール「Chiiin」

わが家の電子レンジが600Wではなく650Wだったため、電子レンジのワット数と時間の計算がすぐにできないのでアプリ化。

アプリURL: <a href="https://nagatani.github.io/Chiiin/" target="_blank">https://nagatani.github.io/Chiiin/</a>

![Chiiinアプリ画面](/images/nagatani.github.io_Chiiin.png)

リポジトリ: [Nagatani/Chiiin](https://github.com/Nagatani/Chiiin)

### 状況説明を図示化するのを助ける作図ツール「こうなってた」

SNSで今朝のねこの状況や、交差点での危ない運転など、状況を説明したいとき、図示できたら良いな。ってなって作った。
作ってる間に状況説明のことをSNSに書くのを忘れるおまけ付き。

アプリURL: <a href="https://nagatani.github.io/kounatteta/" target="_blank">https://nagatani.github.io/kounatteta/</a>

![こうなってたアプリ画面](/images/nagatani.github.io_kounatteta.png)

リポジトリ: [Nagatani/kounatteta: 状況図示を補助するツール](https://github.com/Nagatani/kounatteta)

## 雑なアプリはPWAで公開しとけば良いのでは？

という話でした。