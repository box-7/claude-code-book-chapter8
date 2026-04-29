# repository-structure.md メモ

タスク管理CLIアプリ（`task-cli`）のディレクトリ構成と命名規則を定義したドキュメント。

`src/` は4層構造になっている。`cli/` がコマンド定義と表示、`services/` がビジネスロジック、`storage/` がJSONファイルへの永続化、`types/` が全体共通の型定義。
依存の方向は `cli → services → storage → types` の一方向のみで、逆方向の依存は禁止。

テストは `tests/unit/`・`tests/integration/`・`tests/e2e/` の3種類に分かれていて、`unit/` は `src/` と同じディレクトリ構造を保つルールになっている。

ファイル名はレイヤーごとにサフィックスが決まっている（`TaskManager.ts`・`FileStorage.ts`・`TaskValidator.ts` など）。1ファイル300行以下が推奨で、500行超えたら分割。

`.task/config.json`（GitHubトークンを含む）はgitignore対象。`.task/tasks.json` はチーム共有できるためデフォルトでは除外しない。
