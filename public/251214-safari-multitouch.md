---
title: iOSのSafariにおいて複数の指でtouchstartとtouchendが同時に起こるときイベントが発火しない件(と、その対策)
tags:
  - JavaScript
  - iOS
  - Safari
  - WebKit
private: false
updated_at: '2025-12-14T02:18:41+09:00'
id: bc816d59ca303d2cabc9
organization_url_name: null
slide: false
ignorePublish: false
---

## バグの概要

[Appleのフォーラムに2022年に投稿されている](https://developer.apple.com/forums/thread/711928)現象です。
そのフォーラムの投稿に再現用のCodePenのリンクもあり、現在もiOS18.5、26.2で動作・再現しました。

[https://codepen.io/arisaito/pen/WNzymjv](https://codepen.io/arisaito/pen/WNzymjv)

<p class="codepen" data-height="300" data-default-tab="js,result" data-slug-hash="WNzymjv" data-pen-title="touchstart/end cheker" data-user="arisaito" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
      <span>See the Pen <a href="https://codepen.io/arisaito/pen/WNzymjv">
  touchstart/end cheker</a> by arisaito (<a href="https://codepen.io/arisaito">@arisaito</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
      </p>
      <script async src="https://public.codepenassets.com/embed/index.js"></script>

このデモページでは下部の青色とピンク色の枠内をタッチすると、タッチしている間だけその上に四角が現れるよう、`touchstart`と`touchend`(青枠) / `pointerdown`と`pointerup`(ピンク枠) を使って実装されています。複数の指で同時にタッチしても正しく動作しますが、問題は**片方の指が画面を離れるのと同時に**もう片方の指が画面に触れた場合に後者の`touchstart`イベントが起こらないという点です。何度か試すと比較的かんたんに再現できると思います。
Androidでは起こりません。

筆者が追加検証した結果、この場合には `touchstart` が発生しないだけでなく、
1. `pointerdown` や `click` といった他のタッチ操作と関連するイベントも同様に発火しない
2. その瞬間だけでなく、それ以降その指が関わる `touchend` 等のイベントも一切発火しない
3. `touchstart`イベントが起こらなかった指でも次の同様の現象を引き起こすことができる
    * 例えば右手`touchend`(通常)と左手`touchstart`(発火しない)が同時に起こったあと、その左手の`touchend`(発火しない)と同時に右手をタッチした場合その`touchstart`もまた発火しない...

ということがわかりました。

:::note info
もし特定のiOSバージョンで修正されているなど続報を知っている方がいらっしゃったら教えていただけると助かります！
:::

## workaround?

現状なさそうです。

AIに聞いたり同様の事例を検索したりすると`preventDefault()`とかが関係ありそうな気がしてきますが、関係ありません。タッチ操作に対してイベントが発火しないことが問題なので、イベントのコールバック内でどうにかできるものではなさそうです。

また、`touchend`側のコールバックをどうにかしたら解決できるものでもなさそうです。そもそも`touchend`イベントをlistenしなかったとしても発生しました。

## 筆者が開発しているアプリでの例

筆者は [Falling Nikochan](https://nikochan.utcode.net/) (GitHub: [na-trium-144/falling-nikochan](https://github.com/na-trium-144/falling-nikochan)) というブラウザで動く音楽ゲームを開発しています。
*(もし興味を持っていただけましたらGitHubのスター・ [YouTube:@nikochan144](https://www.youtube.com/channel/UChAEFwUtjsbWmWwZSxYLXWQ) や [X:@nikochan144](https://x.com/nikochan144) のチャンネル登録/フォローをしていただけると嬉しいです！)*

https://nikochan.utcode.net/

参考までに、この問題に対応する [issue #523](https://github.com/na-trium-144/falling-nikochan/issues/523) があります。大したことは書いてません。

このゲームはPCではキーボードで、タブレット・スマホではタッチ操作でプレイすることができます。現状ではこのゲームに登場する音符はすべてキーを押した瞬間(`keydown`)およびタッチした瞬間(`pointerdown`)だけに判定があります。ちなみに最高評価の判定は±40msです。
2本以上の指でタッチする譜面も普通にあるので、運悪く`pointerdown`と`pointerup`が重なると判定されない場合があるというのは音ゲーとしては致命的でした。

また、音ゲーにしては比較的ライトユーザーをターゲットにしていることもあり、アナリティクスを見ると全アクセス数の約3分の1がiOSであるため、ユーザーへの影響もかなり大きいです。

### 暫定的な対策

苦肉の策として、その現象が発生した際にも発火している唯一のイベントである`pointerup`を利用することにしました。
もしある指を離したタイミング(`pointerup`)が次の音符の判定タイミングとぴったり重なっており、かつその音符に対して次の`pointerdown`が検出されなかったのであれば、実はユーザーは正しくタップしているにも関わらずこのバグによりスルーされてしまっている可能性が存在することになります。
そのため、その場合には該当する`pointerup`のタイミングでタップされたものとみなして判定を行う、という仕様にしました。

デメリットとして、このゲームではタップの瞬間にSEが鳴るのですが、あとから`pointerup`のタイミングを参照して判定を行うこの実装であとからSEを鳴らすことはできません。そのためiOSでのプレイ時にはSE全体を無効にせざるを得ませんでした。

また、これをそのまま実装すると 本当にタップしていないものが誤って拾われる・この仕様を知ってしまえば簡単に悪用できてしまう、というのを避けたかったので、その場合には判定を厳しくする(具体的には±25msにする)ということでバランスを取りました。

それと、概要の3.に記載したようにこのバグが2回以上連続して起こった場合は`touchend`も`touchstart`も起こらないので反応できません。さすがにどうしようもない気がします。

*(このような理由から、このゲームで精度狙いをする方にはiOSだけはおすすめしません。)*
