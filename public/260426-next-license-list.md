---
title: Next.jsで利用しているパッケージのライセンス一覧を自動生成して表示する「next-license-list」
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

Next.jsでWebアプリケーションを開発している際、利用しているサードパーティ製パッケージのライセンス一覧（著作権表示）を表示したいことがよくあります。

多くのOSSライセンス（MITやApacheなど）には、**「著作権表示およびライセンス全文を、配布物に含めなければならない」**という条件があります。Webサイトの場合、ブラウザがJSファイルをダウンロードして実行するため、これは「ソフトウェアの配布」に該当します。

手動でリストを作るのは大変ですし、更新を忘れてしまうこともあります。
そこで、ビルド時に自動でライセンス情報を収集し、簡単にアプリケーション内で利用できるようにするライブラリ **`next-license-list`** を作成しました。

## 特徴

- **自動収集**: webpackのビルドプロセスを利用して、実際にバンドルに含まれるパッケージのライセンス情報を収集します。
- **サーバーコンポーネント対応**: App RouterのServer ComponentsやAPI Routesから簡単に情報を取得できます。
- **様々なビルドモードに対応**: 通常のビルドに加え、`output: 'standalone'` や `output: 'export'` (静的書き出し) でも動作します。
- **リポジトリURLの正規化**: `git+https://...` や `github:user/repo` といった様々な形式のURLを、ブラウザで開ける `https://...` 形式に自動で変換します。
- **型安全**: TypeScriptで定義された型が提供されます。

## 使い方

### 1. インストール

```bash
npm install next-license-list
```

### 2. `next.config.mjs` の設定

`withLicense` という高階関数を使って設定をラップします。

> [!WARNING]
> このライブラリは webpack を利用します。Next.js 15 以降などで Turbopack を利用している場合は、ビルド時に `--webpack` オプションを指定するか、Turbopackを無効にする必要があります。

```javascript:next.config.mjs
import { withLicense } from "next-license-list/config";
import { dirname } from "node:path";
import { fileURLToPath } from "node:url";

/** @type {import('next').NextConfig} */
const nextConfig = {
  // 自分の設定
};

export default withLicense(nextConfig, {
  // オプション (webpack-license-plugin のオプションが渡せます)
  // 例: バンドルに含まれない（CSSフレームワークなど）パッケージを明示的に追加する
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

## 便利なユーティリティ

### リポジトリURLの正規化

`package.json` に記載されているリポジトリURLは形式がバラバラなことが多いですが、このライブラリが提供する `normalizeRepositoryURL` を使うと、きれいなHTTPS URLに変換できます。これは `getLicenses()` 内部でも自動的に利用されています。

```ts
import { normalizeRepositoryURL } from "next-license-list";

const entry = {
  name: "some-pkg",
  repository: "git+https://github.com/user/repo.git",
  // ...
};

const normalized = normalizeRepositoryURL(entry);
console.log(normalized.repository); // "https://github.com/user/repo"
```

## なぜこれが必要だったのか

Next.jsでライセンス一覧を表示するための既存の手法には、いくつか課題がありました。

1. **`license-checker` などをビルド前に実行する**:
   - `devDependencies` まで含まれてしまったり、逆にバンドルに含まれるはずのものが漏れたりすることがあります。
   - `pnpm` などの環境で正しくパスを解決するのが面倒なことがあります。

2. **`webpack-license-plugin` を直接使う**:
   - 生成されたJSONファイルをどこに置くか、それをどうやって実行時に読み込むか（特に `standalone` モードの場合）を自分で制御する必要があります。

`next-license-list` は、これらをNext.jsの流儀に合わせてカプセル化しています。ビルド時に生成されたJSONの場所を自動的に管理し、`getLicenses()` というシンプルなAPIでアクセスできるようにしています。

## 仕組み

1. ビルド時（`next build`）に `webpack-license-plugin` が走り、パッケージ情報を収集します。
2. 収集したデータは、プロジェクト内の一時的なディレクトリ（`node_modules/next-license-list/generated/`）にJSONとして保存されます。
3. `getLicenses()` が呼ばれると、そのJSONを読み込んで返します。
4. **Standaloneモードへの対応**: `output: 'standalone'` を利用している場合、Next.jsのトレーサーがこのJSONファイルも依存関係として認識するため、デプロイ先にも自動的に含まれます。

## まとめ

`next-license-list` を使うと、Next.jsアプリケーションにおけるライセンス表示の実装が非常に楽になります。サイトのデザインに合わせて自由にスタイリングできるため、アクセシビリティやブランドイメージを損なうことなく、OSSコンプライアンスを遵守できます。

GitHub: [na-trium-144/next-license-list](https://github.com/na-trium-144/next-license-list)
