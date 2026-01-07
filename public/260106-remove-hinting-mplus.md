---
title: Node.jsでウェブフォントからhintingを削除し、Windowsで生じるジャギーを解消
tags:
  - CSS
  - Node.js
  - font
  - webfont
private: false
updated_at: '2026-01-07T15:43:03+09:00'
id: ef71e065067289550da3
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- ref. https://github.com/ut-code/my-code/pull/156 -->

ウェブフォントの [M PLUS Rounded 1c](https://fonts.google.com/specimen/M+PLUS+Rounded+1c) (Rounded M+ 1c)、これを自分はnpmの `@fontsource/m-plus-rounded-1c` パッケージを使って自分のプロジェクトにダウンロードしてウェブサイトにバンドルしていますが、Windowsにおいて16px以下のフォントサイズで表示した際になめらかに表示されません。(ジャギーというらしい)

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

0.03degの回転は表示上はまったくわからないですが、ページ全体に適用するとレイアウトやテキスト以外の要素の表示に悪影響があったり、不必要なレンダリング負荷の増加が気になります。
テキスト部分だけにCSSを適用するのは、ページ全体でこのフォントが広く使われている場合面倒です。

## 根本原因

16px以下のサイズでアンチエイリアスがかからなくなるのは、フォントのヒンティングによるものです。本来は小さいサイズでフォントをきれいに表示するためのものなはずですが、Rounded M+のヒンティングはWindowsのレンダリングエンジンと相性が悪いらしいです。

したがって、フォントからヒンティング情報を削除してやると解決します。そのためにはFontForgeなどのソフトを使ってフォントデータを編集するのが一般的なやり方のようです。

## Node.jsでヒンティングを削除する

しかし、何MBもする修正後のフォントデータを自分のプロジェクトのリポジトリに直接commitしたくありません。ダウンロードはnpmで行い、プロジェクトのbuild時にフォントデータからヒンティングを削除する処理が行われるようにできたら良さそうです。
Node.jsでフォントファイルを編集する方法を探してみると、 [fonteditor-core](https://www.npmjs.com/package/fonteditor-core) というパッケージがありました。これを使用して `@fontsource/m-plus-rounded-1c` パッケージに修正を加えたものを `./app/m-plus-rounded-1c-nohint` に書き出す、以下のようなスクリプトを作りました。

```ts:removeHinting.ts
import { createFont, woff2 } from "fonteditor-core";
import { existsSync } from "node:fs";
import { mkdir, readdir, readFile, writeFile } from "node:fs/promises";
import path from "node:path";
import pako from "pako";

const fontsPath = "./node_modules/@fontsource/m-plus-rounded-1c/files/";
const cssPath = "./node_modules/@fontsource/m-plus-rounded-1c/";
const outPath = "./app/m-plus-rounded-1c-nohint/";

const weights = [400, 700];

if (existsSync(outPath)) {
  console.log(`Output directory ${outPath} already exists.`);
  console.log("To regenerate font files, please delete the directory.");
} else {
  await woff2.init();
  await mkdir(outPath);

  async function removeHintingFromFont(file: string) {
    const fontBuffer = await readFile(path.join(fontsPath, file));

    const font = createFont(fontBuffer, {
      type: "woff",
      hinting: false,
      kerning: true,
      compound2simple: false,
      inflate: (data) => Array.from(pako.inflate(Uint8Array.from(data))),
    });

    const woffBuffer = font.write({
      type: "woff",
      hinting: false,
      kerning: true,
      deflate: (data) => Array.from(pako.deflate(Uint8Array.from(data))),
    }) as Buffer;
    const outFileName = path.parse(file).name + `-nohint.woff`;
    await writeFile(path.join(outPath, outFileName), woffBuffer).then(() => {
      console.log(`Processed ${file} -> ${outFileName}`);
    });

    const woff2Buffer = font.write({
      type: "woff2",
      hinting: false,
      kerning: true,
    }) as Buffer;
    const outFileName2 = path.parse(file).name + `-nohint.woff2`;
    await writeFile(path.join(outPath, outFileName2), woff2Buffer).then(() => {
      console.log(`Processed ${file} -> ${outFileName2}`);
    });
  }

  async function rewriteCSS(file: string) {
    let css = await readFile(path.join(cssPath, file), "utf-8");
    css = css.replace(/url\((.+?)\)/g, (match, p1) => {
      const parsedPath = path.parse(p1);
      return `url(./${parsedPath.name}-nohint${parsedPath.ext})`;
    });
    css = css.replaceAll(
      "font-family: 'M PLUS Rounded 1c'",
      "font-family: 'M PLUS Rounded 1c NoHint'"
    );
    await writeFile(path.join(outPath, file), css, "utf-8");
    console.log(`Rewritten CSS: ${file}`);
  }

  for (const file of await readdir(fontsPath)) {
    if (
      path.extname(file) === ".woff" &&
      weights.some((w) => file.includes(w.toString()))
    ) {
      await removeHintingFromFont(file);
    }
  }
  for (const file of await readdir(cssPath)) {
    if (
      path.extname(file) === ".css" &&
      weights.some((w) => file.includes(w.toString()))
    ) {
      await rewriteCSS(file);
    }
  }
}
```

そして、package.jsonのscriptsでビルド前にこのスクリプトが実行されるようにします。
```json:package.json
{
  "scripts": {
    "dev": "npm run removeHinting && next dev",
    "build": "npm run removeHinting && next build",
    "removeHinting": "tsx ./scripts/removeHinting.ts",
  }
}
```
tsファイルの実行環境は `tsx` でも `ts-node` でも `node --experimental-strip-types` でも `bun` でも好きなものを使えば良いとおもいます。

### フォントデータの編集

このスクリプトは `./node_modules/@fontsource/m-plus-rounded-1c/files/` 以下にある .woff ファイルを読み込み、hintingを除いたものを `./app/m-plus-rounded-1c-nohint/${元ファイル名}-nohint.woff`, `./app/m-plus-rounded-1c-nohint/${元ファイル名}-nohint.woff2` として書き出します。書き出す際に `hinting: false` を指定すればhinting情報がなくなります。
自分のプロジェクトでは400と700のweightしか使わないので、このスクリプトではファイル名に400か700を含むもののみ編集を行っています。

`kerning: true` は [font-kerning - CSS | MDN](https://developer.mozilla.org/ja/docs/Web/CSS/Reference/Properties/font-kerning) で説明されているようなものです。fonteditor-coreではデフォルトでfalseになっていますが、Rounded M+にはkerning情報があり使ったほうが良いので、残しておきます。

生成されるフォントデータはどの環境で実行しても同一でした。最終的にwebpack等(他のバンドラーでもたぶん同様だと思いますが)でバンドルする際にはファイル名にハッシュが追加されますが、ビルドのたびにハッシュが変わってキャッシュが効かなくなる、といったこともありません。なのでこのスクリプトがプロジェクトのビルドの際に毎回実行されるようにする運用で問題なさそうです。

この変換処理にはおよそ1分弱かかります。開発時に毎回実行していては手間なので、すでに出力ディレクトリが存在した場合はなにもしないようにしています。
```ts
if (existsSync(outPath)) {
  console.log(`Output directory ${outPath} already exists.`);
  console.log("To regenerate font files, please delete the directory.");
} else {
```

### CSSの編集

`./app` 以下に書き出したフォントファイルが使用されるように、`./node_modules/@fontsource/m-plus-rounded-1c/` 以下にある .css ファイルに修正を加えたものを同じディレクトリに書き出しています。具体的にはこのスクリプトによってCSSが以下のように変わります。

```diff_css:400.css
 /* m-plus-rounded-1c-[0]-400-normal */
 @font-face {
-  font-family: 'M PLUS Rounded 1c';
+  font-family: 'M PLUS Rounded 1c NoHint';
   font-style: normal;
   font-display: swap;
   font-weight: 400;
-  src: url(./files/m-plus-rounded-1c-0-400-normal.woff2) format('woff2'), url(./files/m-plus-rounded-1c-0-400-normal.woff) format('woff');
+  src: url(./m-plus-rounded-1c-0-400-normal-nohint.woff2) format('woff2'), url(./m-plus-rounded-1c-0-400-normal-nohint.woff) format('woff');
   unicode-range: U+25ee8,U+25f23,U+25f5c,U+25fd4,U+25fe0,U+25ffb,U+2600c,U+26017,U+26060,U+260ed,U+26222,U+2626a,U+26270,U+26286,U+2634c,U+26402,U+2667e,U+266b0,U+2671d,U+268dd,U+268ea,U+26951,U+2696f,U+26999,U+269dd,U+26a1e,U+26a58,U+26a8c,U+26ab7,U+26aff,U+26c29,U+26c73,U+26c9e,U+26cdd,U+26e40,U+26e65,U+26f94,U+26ff6-26ff8,U+270f4,U+2710d,U+27139,U+273da-273db,U+273fe,U+27410,U+27449,U+27614-27615,U+27631,U+27684,U+27693,U+2770e,U+27723,U+27752,U+278b2,U+27985,U+279b4,U+27a84,U+27bb3,U+27bbe,U+27bc7,U+27c3c,U+27cb8,U+27d73,U+27da0,U+27e10,U+27eaf,U+27fb7,U+2808a,U+280bb,U+28277,U+28282,U+282f3,U+283cd,U+2840c,U+28455,U+284dc,U+2856b,U+285c8-285c9,U+286d7,U+286fa,U+28946,U+28949,U+2896b,U+28987-28988,U+289ba-289bb,U+28a1e,U+28a29,U+28a43,U+28a71,U+28a99,U+28acd,U+28add,U+28ae4,U+28bc1,U+28bef,U+28cdd,U+28d10,U+28d71,U+28dfb,U+28e0f,U+28e17,U+28e1f,U+28e36,U+28e89,U+28eeb,U+28ef6,U+28f32,U+28ff8,U+292a0,U+292b1,U+29490,U+295cf,U+2967f,U+296f0,U+29719,U+29750,U+29810,U+298c6,U+29a72,U+29d4b,U+29ddb,U+29e15,U+29e3d,U+29e49,U+29e8a,U+29ec4,U+29edb,U+29ee9,U+29fce,U+29fd7,U+2a01a,U+2a02f,U+2a082,U+2a0f9,U+2a190,U+2a2b2,U+2a38c,U+2a437,U+2a5f1,U+2a602,U+2a61a,U+2a6b2,U+2a9e6,U+2b746,U+2b751,U+2b753,U+2b75a,U+2b75c,U+2b765,U+2b776-2b777,U+2b77c,U+2b782,U+2b789,U+2b78b,U+2b78e,U+2b794,U+2b7ac,U+2b7af,U+2b7bd,U+2b7c9,U+2b7cf,U+2b7d2,U+2b7d8,U+2b7f0,U+2b80d,U+2b817,U+2b81a,U+2d544,U+2e278,U+2e569,U+2e6ea,U+2f804,U+2f80f,U+2f815,U+2f818,U+2f81a,U+2f822,U+2f828,U+2f82c,U+2f833,U+2f83f,U+2f846,U+2f852,U+2f862,U+2f86d,U+2f873,U+2f877,U+2f884,U+2f899-2f89a,U+2f8a6,U+2f8ac,U+2f8b2,U+2f8b6,U+2f8d3,U+2f8db-2f8dc,U+2f8e1,U+2f8e5,U+2f8ea,U+2f8ed,U+2f8fc,U+2f903,U+2f90b,U+2f90f,U+2f91a,U+2f920-2f921,U+2f945,U+2f947,U+2f96c,U+2f995,U+2f9d0,U+2f9de-2f9df,U+2f9f4;
 }
```

一応改変したフォントであるということがわかるようにFontFamilyを変更しています。

urlについては、cssファイルからの相対パスです。バンドラーがこれを読んだ際にwoffファイルをassetに含め、適切なurlに置き換えてくれます。

最後に`@fontsource/m-plus-rounded-1c`の代わりにこのcssをプロジェクトでimportするようにすればokです。

```tsx:layout.tsx
// import "@fontsource/m-plus-rounded-1c/400.css";
// import "@fontsource/m-plus-rounded-1c/700.css";
import "./m-plus-rounded-1c-nohint/400.css";
import "./m-plus-rounded-1c-nohint/700.css";
```
