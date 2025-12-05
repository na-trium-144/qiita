---
title: "Next.jsで特定のモジュールをサーバーサイドでimportさせない方法"
tags:
  - Next.js
  - React
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

Next.jsで一部のコンポーネントについてSSRを無効化したい場合、next/dynamic が使えます。
参考: [公式ドキュメント](https://nextjs.org/docs/pages/guides/lazy-loading)

主な用途としては、SSRに対応していないライブラリ(サーバーサイドでレンダリングすると Hydration Error を起こしてしまうなど)を使いたい場合が挙げられます。
また、Cloudflare WorkerにNext.jsのアプリをデプロイする際にはサーバーサイドのバンドルサイズを3MB以下に抑える必要があるため、サイズの大きいコンポーネントはサーバーサイドではimportしないようにする(クライアントのアセットサイズは制限がゆるいので、クライアントサイドでimportしてレンダリングする)という用途でも使えると思いました。

しかし、next/dynamicをそのまま使えないケースに2つ遭遇したので、その際に個人的に採用した代案を以下に紹介します。

:::note warn
この記事の内容は Next.js 15.4, `@opennextjs/cloudflare` 1.7.1 を使っていたときのものです。
:::

# Reactコンポーネントでないものをクライアントのみでのimportにしたい場合

[typescript](https://www.npmjs.com/package/typescript) はブラウザでimportすることでTS Playgroundのようなものを作るのに使えますが、とても3MBには収まりませんでした。

```ts
"use client";

import ts from "typescript";
```

typescriptはReactコンポーネントではないので、next/dynamicは使えません。
その代わり、こういう場合は dynamic import をすればサーバーサイドでのインポートを避けることができます。

```ts
useEffect(() => {
  (async () => {
    const ts = await import("typescript");
    // do something
  })();
}, []);
```

...のはずなんですけど、実際にこれをビルドしてみるとなぜかtypescriptモジュールがサーバーサイドのバンドルに含まれてしまいました。

サーバーサイドでは絶対にimportせず、バンドルにそもそも含んでほしくないので、いろいろ試した結果以下のように無意味なif文を追加してやるとうまくいきました。

```ts
useEffect(() => {
  (async () => {
    if (typeof window !== "undefined") {
      const ts = await import("typescript");
      // do something
    }
  })();
}, []);
```

Next.js(が使っているバンドラー)、もしくはそれをCloudflareで動かすために使ったOpenNextのバグですかね。アップデートで今後修正されこのようなif文が不要になる可能性もあるとは思いますが、当面は dynamic import をする際には上記のようなif文を挟むようにします。(入れるデメリットは特になさそうですし)

# Reactコンポーネントではあるけどnext/dynamicでは機能不足な場合

例えば [react-syntax-highlighter](https://github.com/react-syntax-highlighter/react-syntax-highlighter) を以下のようにnext/dynamicに入れたとします。

```tsx
"use client";

import dynamic from "next/dynamic";
// import SyntaxHighlighter from "react-syntax-highlighter";
const SyntaxHighlighter = dynamic(
  async () => {
    // 挙動をわかりやすくするために遅延を入れている
    await new Promise((r) => setTimeout(() => r(undefined), 1000));
    return await import("react-syntax-highlighter");
  },
  {
    ssr: false,
    loading: () => <div>Loading...</div>,
  },
);

export default function Home() {
  return (
    <main className="">
      コードの前の文章
      <SyntaxHighlighter language="javascript">
        {'console.log("hello, world");\n'.repeat(20)}
      </SyntaxHighlighter>
      コードのあとの文章
    </main>
  );
}
```

この場合、ページの読み込み直後にはSyntaxHighlighterの部分には「Loading...」が表示され、

![before-loading.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4275950/d4a6338c-5759-443e-88a0-eb679e2b3bcb.png)

SyntaxHighlighterのモジュールのimportが完了したらSyntaxHighlighterで置き換えられます。
SyntaxHighlighter内のコードが長い場合、この読み込み前後でレイアウトが大きく動いてしまい、ユーザーエクスペリエンス的に良くありません。

![after-loading.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4275950/dc5563fa-ebeb-41d2-aeb5-cd5643622f8c.png)

読み込みが完了するまでシンタックスハイライトができないのは仕方ないとして、シンタックスハイライトなしのコードでも表示してレイアウトをできるだけ保ちたいものです。

```tsx
const SyntaxHighlighter = dynamic(
  async () => {
    // 挙動をわかりやすくするために遅延を入れている
    await new Promise((r) => setTimeout(() => r(undefined), 1000));
    return await import("react-syntax-highlighter");
  },
  {
    ssr: false,
    loading: (props) => <pre>{props.children}</pre>, // これはできない
  },
);
```

しかし、next/dynamicのloadingオプションにはこのようにpropsを渡す機能はありませんでした。

そのためnext/dynamicを使わずにコンポーネントを遅延読み込みする必要があります。next/dynamicのドキュメントにはnext/dynamicは[React.lazy()](https://react.dev/reference/react/lazy)と[Suspense](https://react.dev/reference/react/Suspense)を組み合わせたものであると書いてあるので、それを組み合わせて自前実装すればよさそうです。

ただし、普通にlazyとSuspenseを組み合わせるだけではSSR中にimportしてしまいます。サーバーサイドでは絶対にSyntaxHighlighterをimportせず常にfallbackを表示するよう、useStateやuseEffectなどを使ってレンダリングを分岐させる必要があります。

```tsx
"use client";

import { lazy, Suspense, useEffect, useState } from "react";
const SyntaxHighlighter = lazy(async () => {
  if (typeof window !== "undefined") {
    await new Promise((r) => setTimeout(() => r(undefined), 1000));
    return await import("react-syntax-highlighter");
  }
  return null!;
});

function FallbackCodeBlock({ children }: { children: string }) {
  // SyntaxHighlighterとCSSレイアウトをできるだけ合わせる
  return <pre style={{ padding: "0.5em" }}>{children}</pre>;
}
export default function Home() {
  const code = 'console.log("hello, world");\n'.repeat(20);
  const [initHighlighter, setInitHighlighter] = useState(false);
  useEffect(() => {
    // クライアントサイドでstateを切り替えることでSyntaxHighlighterのimportを開始する
    setInitHighlighter(true);
  }, []);
  return (
    <main className="">
      コードの前の文章
      {initHighlighter ? (
        <Suspense
          fallback={
            /* クライアントサイドでimportが完了するまでの間Fallbackを表示 */
            <FallbackCodeBlock>{code}</FallbackCodeBlock>
          }
        >
          <SyntaxHighlighter language="javascript">{code}</SyntaxHighlighter>
        </Suspense>
      ) : (
        /* SSR時: initHighlighterがfalseで、importを試行せずfallbackを表示 */
        <FallbackCodeBlock>{code}</FallbackCodeBlock>
      )}
      コードのあとの文章
    </main>
  );
}
```

