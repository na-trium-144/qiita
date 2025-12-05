# qiita

```bash
npm ci
npx qiita login
npx qiita preview
```

### 記事の作成

```bash
npx qiita new yymmdd-hogefuga
```

下書きは draft/yymmdd-hogefuga ブランチに、書き上げたらmainへ

2記事以上を同じブランチで編集して同時にmainへマージすると、CIが403Forbiddenで落ちる。なんでやねん。
