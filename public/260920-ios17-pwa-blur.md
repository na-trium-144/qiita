---
title: iOS 27 PWAで画面上端に表示されるブラーを消す方法
tags:
  - iOS
  - Safari
  - PWA
  - CSS
  - WebKit
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

iOS 27 以降、PWA（ホーム画面に追加したWebアプリ）において画面上端に意図しないプログレッシブブラー（ぼかし効果）がオーバーレイ表示されてしまうという現象が発生します。
この挙動は、アプリ全体を `height: 100dvh` などで固定し、ツールバーやステータスバーと被るコンテンツが存在しない画面設計であっても発生します。

<img width="300" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4275950/780c50d1-94fb-46e2-b17e-8f7376a3c643.png">

:::note
ちなみに画像は筆者が開発しているブラウザで動く音楽ゲーム [Falling Nikochan](https://nikochan.utcode.net/) です。
:::


この画像のように、画面上端に `position: fixed;` で一定以上の高さを持つダミーの要素があれば、上端のブラーが消えます。
(この画像では誇張して前面かつ必要以上に大きめに配置しています)

<img width="300" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4275950/c309dea7-d0ef-4d7a-9d5b-398312fd0c5a.png">

さらに、CSSの **`background-clip: text`** を利用してこのダミー要素を非表示にすることで、画面の見た目や一切邪魔することなくこのブラーだけ消すことができます。

<img width="300" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4275950/66d94b09-6603-4ba5-b2b6-16ad4eeb2350.png">


## 解決用のCSSスニペット

```html
<div class="ios-blur-fix" aria-hidden="true"></div>
```

```css
.ios-blur-fix {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 11px; /* WebKitの判定条件として10pxを超える必要がある */
  z-index: 2147483647; /* 0以上であれば有効 */
  pointer-events: none; /* タップやスクロールなどの操作を邪魔しない */
  background-color: #ffffff; /* この色がiOSのステータスバー領域に反映される */
  -webkit-background-clip: text;
  background-clip: text; /* テキスト（空）にクリップすることで画面上には描画されない */
}
```

### 各プロパティのポイント
- `height: 11px`: 後述するWebKitの判定ロジックを満たすため、一定以上のサイズが必要です。11px以上あれば確実のようです。
- `pointer-events: none`: 画面最前面に置いても背後のボタンやリンクのクリック・タップを邪魔しません。
- `background-color`: ここに指定した背景色がiOSのステータスバー領域の背景として認識・反映されます。
- `background-clip: text`（最重要）: 背景をテキスト領域だけにクリップします。要素内にテキストが存在しないため、**ブラウザ上にはピクセルとして一切描画されません（透明になります）**。


## なぜこれで消えるのか？（WebKitの判定ロジック）

この現象の原理やWebKitの詳細な判定条件については、先行して調査された以下の記事で詳しく解説されています。

https://zenn.dev/uakihir0/articles/260920-ios27-pwa-blur

WebKitは画面上端に固定ヘッダーが存在するかどうかを判定し、存在しないと判断した場合に「コンテンツがステータスバー裏に潜り込むスクロール領域である」とみなして自動的にプログレッシブブラーを付加します。

この判定において重要なのは、**実際に画面へレンダリングされたピクセルではなく、CSSのスタイル設定を基準に判定されている**という点です。

**`background-clip: text;`** は、CSSのプロパティ上は有効な背景色（`background-color`）とサイズを持った `position: fixed` 要素として認識され、「WebKitには上端に固定要素が存在すると認識させつつ、画面上には何も描画しない」という理想的なダミー要素を作ることができます。

:::note warn
`opacity: 0` や `visibility: hidden`, `display: none` などの方法で非表示にしようとすると、WebKitの判定条件から外れてしまいブラーが復活してしまうため、ワークアラウンドにはなりません。
:::

## iOS Safariのツールバー/ステータスバー透過仕様との共通点

この判定ロジックは、iOS 26以降のSafariにおける「ツールバー・ステータスバーの色設定・透過挙動」とほぼ共通しています。

Safariのバー透過や色変化に関しては、これまでにもいくつかの記事で調査されています。

https://zenn.dev/timelab/articles/b471d6beed9f6e

https://qiita.com/kskwtnk/items/df4d6b15f6df7026cfeb

https://zenn.dev/tsunagu/articles/cc8f93dfe12759

上下端に実際に背景色を持つ固定ヘッダーやフッターを置くことでツールバーやステータスバーを非透過にすることができますが、今回の `background-clip: text` を用いた手法は、
「固定ヘッダー・フッターでコンテンツの上下を隠すことなく、ツールバーやステータスバーを非透過にしたい」
というケースにそのまま転用可能です。

### SafariとPWAにおける挙動の整理

条件を満たす固定要素（`position: fixed`）の有無によって、iOS 26/27のSafariおよびiOS 27のPWAは以下のように振る舞います。

| 状態 | iOS 26 / 27 Safari | iOS 27 PWA |
| :--- | :--- | :--- |
| 条件を満たす `position: fixed` 要素がある場合 | バーが**非透過**になり、その要素の色で塗られる | 上端の**ブラーが消え**、その要素の色でステータスバーが塗られる |
| 該当する固定要素がない場合 | バーが**透過**し、コンテンツが透ける | ツールバーは `:root` やシステムの背景色になり、**プログレッシブブラーが強制付与される** |

SafariでもPWAでも、透過のケースではツールバーは `:root` の背景色にフォールバックし、それも指定されていない場合に白などシステムの背景色になるようです。

:::note warn
プログレッシブブラーの付与は iOS 26 PWA にはなかった挙動です。
:::

## まとめ

- iOS 27 PWAで発生する画面上端の強制ブラーは、iOS 26から続くWebKitの「画面端の固定要素判定」の延長線上にある仕様と考えられます。
- `opacity: 0` では判定から除外されてしまいますが、 `background-clip: text` を使うことで、見た目に影響を与えずにWebKitの判定述語だけを満たすことができます。
- 現状、PWAのUIデザインを崩さずにブラーを消す最も副作用の少ないワークアラウンドです。同様の現象に悩まされている方はぜひ試してみてください。
