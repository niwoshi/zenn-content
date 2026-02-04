---
title: "はじめてのGitHub Actions入門"
emoji: "🚀"
type: "tech"
topics: ["github", "githubactions", "ci", "automation"]
published: false
---

## はじめに

GitHub Actionsは、GitHubが提供するCI/CDツールです。この記事では、GitHub Actionsの基本的な使い方を解説します。

## GitHub Actionsとは

GitHub Actionsを使うと、以下のようなことが自動化できます。

- コードのテスト実行
- ビルド・デプロイ
- コードのフォーマットチェック
- 定期的なタスクの実行

## 基本的なワークフローの書き方

`.github/workflows/`ディレクトリにYAMLファイルを配置することで、ワークフローを定義できます。

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test
```

## ワークフローの構成要素

### トリガー（on）

ワークフローがいつ実行されるかを指定します。

| トリガー | 説明 |
|---------|------|
| push | プッシュ時に実行 |
| pull_request | PR作成・更新時に実行 |
| schedule | 定期実行（cron形式） |
| workflow_dispatch | 手動実行 |

### ジョブ（jobs）

ワークフロー内で実行される一連のステップをまとめたものです。複数のジョブは並列で実行されます。

### ステップ（steps）

ジョブ内で順番に実行される個々のタスクです。

## まとめ

GitHub Actionsを使えば、開発ワークフローを簡単に自動化できます。まずは簡単なテスト実行から始めてみましょう。

## 参考リンク

- [GitHub Actions公式ドキュメント](https://docs.github.com/ja/actions)


![](https://pub-a8d4733b815843debe800c5f6c36a2ef.r2.dev/ShareX/2026/02/Cursor_Ye8ssU7tVP.png)
