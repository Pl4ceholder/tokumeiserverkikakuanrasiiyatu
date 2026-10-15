# 匿名Terrariaサーバー 説明ページ

- `index.html`：そのままブラウザで開ける、単一ファイルの静的説明ページ
- `anonymous_terraria_server_spec.md`：統合仕様書の原文

## ローカルで確認
`index.html` をブラウザで開くだけで表示できます。外部ライブラリ、画像CDN、ビルド工程は不要です。ミッション候補検索とカテゴリ絞り込みもページ内JavaScriptで動きます。

## 静的ホスティングで公開

### GitHub Pages（例）
1. 新しいGitHubリポジトリを作り、`index.html` をリポジトリ直下に配置します。
2. Settings → Pages を開き、公開元を `Deploy from a branch`、ブランチを `main`、フォルダを `/ (root)` に設定します。
3. 保存すると、Pagesに表示されるURLで公開できます。

### Netlify / Cloudflare Pages
このフォルダの中身を静的サイトとしてアップロードするだけで公開できます。ビルドコマンドは不要です。

## 更新時
説明内容を変更したら `index.html` を更新して再アップロードしてください。元の仕様案は同梱のMarkdownを参照します。バージョン表記は仕様作成時点のもので、実装着手時には公式TShockリリースで再確認する前提です。
