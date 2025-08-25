---
id: 1odn297tlhg
title: GitHub Actions を使って Markdown ドキュメントを GitHub Pages に公開する手順
date: "2025-08-25"
updated: "2025-08-25"
category: blog
thumbnail: ""
tags: [GitHub Actions, GitHub Pages, Jekyll]
slug: blog/2025-08-25-GitHubActionsを使ってMarkdownドキュメントをGitHubPagesに公開する手順
lang: ja
---

GitHub Pages は，静的サイトを簡単にホスティングできる便利な機能です．特に，プロジェクトのドキュメントを Markdown で管理している場合，GitHub Actions を活用することで，自動的にビルド・デプロイが可能になります．本記事では，GitHub Actions を用いて Markdown ドキュメントを GitHub Pages に公開する手順を解説します．

---

## 1. GitHub Pages の設定

まずは，リポジトリの **Settings** に移動します．
左側メニューの **Pages** を選択し，**Build and deployment** の **Source** を **GitHub Actions** に設定します．
この設定により，GitHub Actions のワークフローを使った自動デプロイが有効になります．

## 2. Jekyll を使った GitHub Pages の構築

GitHub Pages はデフォルトで Jekyll をサポートしています．Markdown ファイルを自動的に HTML に変換し，公開することが可能です．
ここでは，GitHub Actions 上で Jekyll を使ってビルドするための設定ファイルを作成します．

## 3. ワークフローファイルの作成

リポジトリ内に `.github/workflows/jekyll-gh-pages.yml` というファイルを作成し，以下のように設定します．

```yaml
name: Deploy Jekyll site to Pages

on:
  push:
    branches: ["main"]
    paths:
      - "README.md"
      - "docs/**"
      - ".github/workflows/jekyll-gh-pages.yml"

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.1"

      - name: Install dependencies
        run: |
          bundle install

      - name: Build site with Jekyll
        run: |
          bundle exec jekyll build

      - name: Deploy to GitHub Pages
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./_site

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build-and-deploy
    steps:
      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v4
```

## 4. paths の設定で無駄なデプロイを防ぐ

上記設定の `paths:` に注目してください．

```yaml
paths:
  - "README.md"
  - "docs/**"
  - ".github/workflows/jekyll-gh-pages.yml"
```

これにより，`README.md`，`docs` ディレクトリ以下のファイル，またはワークフロー自体に変更があった場合のみデプロイが実行されます．
他のファイル変更時にはワークフローが無駄に走らないため，効率的な CI/CD を実現できます．

## 5. デプロイの確認

設定をプッシュすると，自動的に GitHub Actions が実行され，ビルドが成功すると GitHub Pages 上でサイトが公開されます．
リポジトリの **Settings → Pages** から公開 URL を確認できます．

## まとめ

- **GitHub Actions と GitHub Pages を組み合わせることで，自動デプロイ可能なドキュメントサイトを簡単に構築できる**
- **`paths` オプションで不要なデプロイを防止することが重要**
- **Jekyll のビルド環境を Actions 上で設定し，`_site` を GitHub Pages にアップロードする**
