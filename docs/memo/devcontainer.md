# devcontainer.json メモ

`.devcontainer/devcontainer.json` は VS Code / GitHub Codespaces でコンテナ開発環境を定義するファイル。

## やっていること

| 設定 | 内容 |
|------|------|
| `image` | Debian Bookworm ベースの公式 devcontainer イメージを使用 |
| `workspaceFolder` | コンテナ内のワーキングディレクトリを `/workspaces/claude-code-book-chapter8` に設定 |
| `features` (node) | Node.js LTS を自動インストール |
| `features` (claude-code) | Anthropic 公式の Claude Code を自動インストール |
| `postCreateCommand` | コンテナ作成後に `npm install` を自動実行 |

## ポイント

- コンテナを起動するだけで Node.js と Claude Code が使える状態になる
- `ghcr.io/anthropics/devcontainer-features/claude-code:1.0` は Anthropic が公式提供する devcontainer Feature
- `npm install` は初回のみ実行される（コンテナ再作成時）
