---
title: Node.jsでウェブフォントからhintingを削除し、Windowsで生じるジャギーを解消
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- ref. https://github.com/ut-code/my-code/pull/156 -->

ウェブフォントの [M PLUS Rounded 1c](https://fonts.google.com/specimen/M+PLUS+Rounded+1c) (Rounded M+ 1c)、Windowsにおいて16px以下のフォントサイズで表示した際にギザギザ(ジャギーというらしい)が生じます。
Mac、iOS、Linux、Androidではこんなことにはならず、なめらかです。

![mplusrounded1c-14px-4x.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4275950/72d84372-36fe-4f04-a995-30a8225269e5.png)
(このスクショはピクセルを見やすくするため補間なしで4倍に拡大しています)

## CSSでの対処法

この現象について検索してみると、CSSのrotateを使うと解消すると書かれた記事がいくつか見つかります。
```css
.text {
  transform: rotate(0.03deg);
}
```
実際にやってみると、確かに0.03deg以上回転するとジャギーは解消します。記事によっては0.04deg、0.05deg(、環境やCSSの条件によってはもっと)の回転が必要と書かれているものもありました。

## 根本原因
