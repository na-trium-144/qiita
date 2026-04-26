---
title: Next.jsで利用しているパッケージのライセンス一覧を自動生成して表示する「next-license-list」
tags:
  - license
  - webpack
  - OSS
  - Next.js
private: false
updated_at: '2026-04-26T20:11:56+09:00'
id: eb84b08a9bfe8affb09e
organization_url_name: null
slide: false
ignorePublish: false
---

Next.jsなどを使った現代のWebアプリケーション開発では、自分で書いたコードだけではなく、膨大な数のサードパーティ製ライブラリを利用することになります。npmでインストールすパッケージの多くは、MITやApacheなどのOSSライセンスが使われており、 **「著作権表示およびライセンス全文を、配布物に含めなければならない」** という条件が含まれています。

Webサイトの場合、ブラウザがJSファイルをダウンロードして実行するため、これは「ソフトウェアの配布」に該当します。したがって、厳密な法的な観点では、Webサイトに使用したすべてのnpmパッケージの著作権表示とライセンス全文を列挙し、それをWebサイト上に表示しユーザーが確認できる状態にする必要があります。

しかし、手動でnpmパッケージのリストを作りライセンス本文を転記し、また忘れずに更新するというのは非現実的です。Next.js側にもこれを自動でやってくれる機能は用意されていません。(ちなみにDiscussionはありました: [vercel/next.js#33588](https://github.com/vercel/next.js/discussions/33588))

そこで、ビルド時に自動でライセンス情報を収集し、簡単にアプリケーション内で利用できるようにするライブラリ **`next-license-list`** を作成しました。

https://github.com/na-trium-144/next-license-list

https://www.npmjs.com/package/next-license-list

## 特徴

### 既存ツールの課題

npmパッケージのライセンス一覧を自動で取得するツールはすでにいくつか存在します。

例えば [license-checker](https://www.npmjs.com/package/license-checker) はプロジェクト全体で使用しているnpmパッケージのライセンスをすべて列挙します。ただし、前述の目的でWebサイトに使用しているライセンスを表示する場合、必要なのはブラウザが実行するバンドルに含まれるものだけであり、devDependenciesやサーバー側でしか実行しないモジュールは不要です。

[license-webpack-plugin](https://www.npmjs.com/package/license-webpack-plugin) や [webpack-license-plugin](https://www.npmjs.com/package/webpack-license-plugin) を使用すると、webpackのバンドルの過程で列挙が行われるので、実際にバンドルに含まれるライセンスのみを列挙することができます。ただし、Webpackのオプション(特に生成するファイルのパスなど)をNext.jsのバンドルプロセスの仕様に合わせて調整する必要があります。さらに、生成したライセンスのリストをどうやってWebサイト内に読み込んで表示するかを自分で制御する必要があります。

`next-license-list` は、これらをNext.jsの流儀に合わせてカプセル化しています。ビルド時に生成されたJSONの場所を自動的に管理し、`getLicenses()` というシンプルなAPIでアクセスできるようにしています。

### リポジトリURLの正規化

`package.json` に記載されているリポジトリURLは形式が決まっていません。 `https://github.com/user/repo` のような HTTPS URL だけでなく、`git://github.com/user/repo`, `git+https://github.com/user/repo`, `git@github.com:user/repo` など、パッケージごとに記述がバラバラです。

人間が読む場合はいずれの形式でも理解できますが、Webサイト内にリンクとして表示したい場合はすべて HTTPS URL に揃える必要があります。 `next-license-list` は取得したパッケージ情報の中のリポジトリURLを自動でHTTPSに修正します。

## 使い方

Next.js 13, 14, 15, 16 の App Router で動作確認しています。

:::note warn
このライブラリは webpack を利用します。Next.js 16 以降ではビルド時に `--webpack` オプションを指定してTurbopackを無効にする必要があります。
:::

### 1. インストール

```bash
npm install next-license-list
```

### 2. `next.config.mjs` の設定

`withLicense` という高階関数を使って設定をラップします。

オプションについては [webpack-license-plugin](https://www.npmjs.com/package/webpack-license-plugin) を参照してください。

```javascript:next.config.mjs
import { withLicense } from "next-license-list/config";
import { dirname } from "node:path";
import { fileURLToPath } from "node:url";

const nextConfig = {
  // 自分の設定
};

export default withLicense(nextConfig, {
  // オプション (webpack-license-plugin のオプションが渡せます)
  // 例: webpack-license-pluginが自動でリストに含まないパッケージ（CSSフレームワークなど）を明示的に追加する
  includePackages: () => [
    dirname(fileURLToPath(import.meta.resolve("tailwindcss/package.json"))),
  ],
});
```

### 3. ライセンス情報の表示

`getLicenses()` を呼び出すだけで、収集されたライセンス情報の配列を取得できます。

```tsx:app/about/licenses/page.tsx
import { getLicenses } from "next-license-list";

export default async function LicensePage() {
  const licenses = await getLicenses();

  return (
    <main>
      <h1>Licenses</h1>
      <ul>
        {licenses.map((entry) => (
          <li key={`${entry.name}@${entry.version}`}>
            <h2>{entry.name} ({entry.version})</h2>
            <p>License: {entry.license}</p>
            {entry.repository && (
              <p>Repository: <a href={entry.repository} target="_blank" rel="noopener noreferrer">{entry.repository}</a></p>
            )}
            <pre style={{ whiteSpace: 'pre-wrap', background: '#f5f5f5', padding: '1rem' }}>
              {entry.licenseText}
            </pre>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

## 仕組み

Next.jsのビルド(`next build`)は以下の流れで行われます。

1. クライアント側のJSバンドルを生成
    - next-license-list はこのときに `webpack-license-plugin` を使用してパッケージ情報を収集します。
    - この間ライセンスリストはまだ生成されていないので、クライアント側のJSバンドルにライセンスリストを含めることができません。したがって、クライアントコンポーネントから `getLicenses()` を呼ぶことはできません。
    - ライセンスリストはプロジェクト内の一時的なディレクトリ(`node_modules/next-license-list/generated/`)にJSONとして出力されます。
        - `import.meta.url` からパス解決しているので、`node_modules`直下にインストールされている場合以外でも動作するはずです。 npm と pnpm で動作確認しています。
2. サーバー側のJSバンドルを生成
    - 1, 2 はおそらく同時並行で行われます。
    - この間もライセンスリストはまだ生成されていない場合があるので、サーバー側のJSバンドルにもライセンスリストを含めることができません。
    - そこで、webpackのconfigの `externals` オプションで、ライセンスリストのファイルについてはバンドルせずもとのimportパスを残すよう指定しています。
3. それを使って (force-dynamicでないルートの) ページをレンダリングする
    - この時点ではすでにライセンスリストが存在するので、サーバーコンポーネントから `getLicenses()` が呼ばれた場合ライセンスリストのimportは成功します。
    - `output: "export"` の場合であってもサーバーコンポーネントからであれば `getLicenses()` が使用できます。
4. (`output: "standalone"` の場合) Next.jsのトレーサーが `.next/standalone` ディレクトリに依存関係をコピーします。
    - 2でバンドルから除外したライセンスリストのJSONファイルも依存関係として認識され、`.next/standalone` 内に自動的に含まれます。
    - したがってこれ以降は一時的な生成ファイル(`node_modules/next-license-list/generated/`)が削除されても動作します。

## まとめ

`next-license-list` を使うと、Next.jsアプリケーションにおけるライセンス表示の実装が非常に楽になります。サイトのデザインに合わせて自由にスタイリングできるため、アクセシビリティやブランドイメージを損なうことなく、OSSコンプライアンスを遵守できます。

気に入っていただけたらGitHubのスターをお願いします！✨

https://github.com/na-trium-144/next-license-list

