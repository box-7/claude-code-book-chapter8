# リポジトリ構造定義書 (Repository Structure Document)

## プロジェクト構造

```
task-cli/
├── src/                        # ソースコード
│   ├── cli/                    # CLIレイヤー（コマンド定義・表示）
│   ├── services/               # サービスレイヤー（ビジネスロジック）
│   ├── storage/                # ストレージ抽象化レイヤー
│   ├── types/                  # 型定義・インターフェース
│   ├── validators/             # 入力バリデーション
│   └── index.ts                # エントリーポイント
├── tests/                      # テストコード
│   ├── unit/                   # ユニットテスト
│   ├── integration/            # 統合テスト
│   └── e2e/                    # E2Eテスト
├── docs/                       # プロジェクトドキュメント
├── .steering/                  # 作業単位のステアリングファイル
├── .task/                      # タスクデータ（実行時生成、gitignore）
├── dist/                       # ビルド成果物（gitignore）
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── eslint.config.js
├── .prettierrc
└── .gitignore
```

---

## ディレクトリ詳細

### `src/cli/` — CLIレイヤー

**役割**: Commander.jsへのコマンド登録、ユーザー入力のパース、結果の整形・表示

**配置ファイル**:
- `TaskCLI.ts`: Commander.jsプログラムのセットアップ、全コマンドの登録
- `formatters/`: 表示整形ロジック（テーブル・カラー出力）
  - `TaskFormatter.ts`: タスク一覧・詳細の表示整形

**命名規則**:
- コマンドクラス: `PascalCase` + `CLI` サフィックス（例: `TaskCLI.ts`）
- フォーマッタクラス: `PascalCase` + `Formatter` サフィックス

**依存関係**:
- 依存可能: `services/`, `types/`, `validators/`
- 依存禁止: `storage/`（データレイヤーへの直接アクセス禁止）

**例**:
```
src/cli/
├── TaskCLI.ts
└── formatters/
    └── TaskFormatter.ts
```

---

### `src/services/` — サービスレイヤー

**役割**: タスク管理・Git操作・GitHub連携のビジネスロジック

**配置ファイル**:
- `TaskManager.ts`: タスクのCRUD・ステータス管理・検索
- `GitService.ts`: Gitブランチの作成・切り替え・ブランチ名生成
- `GitHubService.ts`: GitHub Issues API・PR作成・同期

**命名規則**:
- サービスクラス: `PascalCase` + `Service` または `Manager` サフィックス

**依存関係**:
- 依存可能: `storage/`, `types/`, `validators/`
- 依存禁止: `cli/`（UIレイヤーへの逆依存禁止）

**例**:
```
src/services/
├── TaskManager.ts
├── GitService.ts
└── GitHubService.ts
```

---

### `src/storage/` — ストレージ抽象化レイヤー

**役割**: データアクセスの抽象化、JSONファイルへの永続化、バックアップ管理

**配置ファイル**:
- `IStorage.ts`: ストレージインターフェース定義（将来のSQLite移行を考慮）
- `FileStorage.ts`: JSONファイルへの読み書き・バックアップ実装

**命名規則**:
- インターフェース: `I` プレフィックス + `PascalCase`（例: `IStorage.ts`）
- 実装クラス: `PascalCase` + `Storage` サフィックス（例: `FileStorage.ts`）

**依存関係**:
- 依存可能: `types/`
- 依存禁止: `services/`, `cli/`（上位レイヤーへの依存禁止）

**例**:
```
src/storage/
├── IStorage.ts
└── FileStorage.ts
```

---

### `src/types/` — 型定義・インターフェース

**役割**: プロジェクト全体で使用するTypeScript型定義の集約

**配置ファイル**:
- `Task.ts`: `Task`, `TaskStatus`, `TaskPriority`, `StatusChange` 型定義
- `Config.ts`: `Config` 型定義
- `Storage.ts`: `StorageData` 型定義
- `errors.ts`: カスタムエラークラス（`AppError`, `ValidationError`, `NotFoundError`）

**命名規則**:
- エンティティ型ファイル: `PascalCase`（例: `Task.ts`）
- エラー定義: `camelCase`（例: `errors.ts`）

**依存関係**:
- 依存可能: なし（他のレイヤーに依存しない純粋な型定義）

**例**:
```
src/types/
├── Task.ts
├── Config.ts
├── Storage.ts
└── errors.ts
```

---

### `src/validators/` — 入力バリデーション

**役割**: CLIから受け取った入力値の検証ロジック

**配置ファイル**:
- `TaskValidator.ts`: タイトル長・日付形式・優先度値域のバリデーション関数

**命名規則**:
- バリデータクラス/ファイル: `PascalCase` + `Validator` サフィックス

**依存関係**:
- 依存可能: `types/`
- 依存禁止: `services/`, `storage/`, `cli/`

---

### `tests/` — テストコード

#### `tests/unit/`

**役割**: 単一クラス・関数のユニットテスト（外部依存はモック）

**構造**: `src/` と同じディレクトリ構造を維持

```
tests/unit/
├── services/
│   ├── TaskManager.test.ts
│   └── GitService.test.ts
├── storage/
│   └── FileStorage.test.ts
└── validators/
    └── TaskValidator.test.ts
```

**命名規則**: `[テスト対象ファイル名].test.ts`

#### `tests/integration/`

**役割**: 複数コンポーネントの連携・実際のファイルシステムを使用したテスト

```
tests/integration/
├── task-crud.test.ts           # タスクのCRUD操作
└── task-workflow.test.ts       # add→start→done の一連フロー
```

#### `tests/e2e/`

**役割**: CLIコマンドを実際に実行するエンドツーエンドテスト

```
tests/e2e/
├── basic-workflow.test.ts      # 基本操作フロー
└── git-integration.test.ts    # Git連携フロー（Gitリポジトリあり/なし）
```

---

### `docs/` — プロジェクトドキュメント

**配置ドキュメント**:
- `product-requirements.md`: プロダクト要求定義書（PRD）
- `functional-design.md`: 機能設計書
- `architecture.md`: 技術仕様書
- `repository-structure.md`: リポジトリ構造定義書（本ドキュメント）
- `development-guidelines.md`: 開発ガイドライン
- `glossary.md`: ユビキタス言語定義（用語集）
- `ideas/`: 壁打ち・アイデアメモ（下書き）

---

### `.steering/` — 作業単位のステアリングファイル

**役割**: 特定の開発作業における「今回何をするか」を定義

**構造**:
```
.steering/
└── 20250115-add-user-authentication/
    ├── requirements.md     # 今回の作業の要求内容
    ├── design.md           # 変更内容の設計
    └── tasklist.md         # タスクリスト
```

**命名規則**: `YYYYMMDD-kebab-case-task-name` 形式（例: `20250115-add-git-integration`）

---

## ファイル配置規則

### ソースファイル

| ファイル種別 | 配置先 | 命名規則 | 例 |
|------------|--------|---------|-----|
| CLIコマンドクラス | `src/cli/` | `PascalCase + CLI.ts` | `TaskCLI.ts` |
| 表示フォーマッタ | `src/cli/formatters/` | `PascalCase + Formatter.ts` | `TaskFormatter.ts` |
| ビジネスロジック | `src/services/` | `PascalCase + Service.ts` または `Manager.ts` | `TaskManager.ts` |
| ストレージ実装 | `src/storage/` | `PascalCase + Storage.ts` | `FileStorage.ts` |
| 型定義 | `src/types/` | `PascalCase.ts` | `Task.ts` |
| バリデーション | `src/validators/` | `PascalCase + Validator.ts` | `TaskValidator.ts` |
| エントリーポイント | `src/` | `index.ts` | `index.ts` |

### テストファイル

| テスト種別 | 配置先 | 命名規則 | 例 |
|-----------|--------|---------|-----|
| ユニットテスト | `tests/unit/[srcと同構造]/` | `[対象].test.ts` | `TaskManager.test.ts` |
| 統合テスト | `tests/integration/` | `[機能]-[種類].test.ts` | `task-crud.test.ts` |
| E2Eテスト | `tests/e2e/` | `[シナリオ].test.ts` | `basic-workflow.test.ts` |

### 設定ファイル（プロジェクトルート）

| ファイル | 用途 |
|---------|------|
| `package.json` | 依存関係・scripts定義 |
| `tsconfig.json` | TypeScriptコンパイル設定 |
| `vitest.config.ts` | Vitestテスト設定 |
| `eslint.config.js` | ESLint設定 |
| `.prettierrc` | Prettier設定 |
| `.gitignore` | Git除外設定 |

---

## 命名規則

### ディレクトリ名
- **レイヤーディレクトリ**: 複数形・`kebab-case`（例: `services/`, `validators/`）
- **機能サブディレクトリ**: 単数形・`kebab-case`（例: `formatters/`）

### ファイル名
- **クラスファイル**: `PascalCase` + 役割サフィックス（`Service`, `Manager`, `Formatter`, `Validator`, `Storage`）
- **型定義ファイル**: `PascalCase`（エンティティ名に合わせる）
- **ユーティリティ・定数**: `camelCase` または `kebab-case`
- **テストファイル**: `[対象ファイル名].test.ts`

---

## 依存関係のルール

### レイヤー間の依存方向

```
src/cli/          →  src/services/  →  src/storage/
       ↘                           ↗
        src/types/  src/validators/
```

**禁止される依存**:
- `src/storage/` → `src/services/` （❌ データレイヤーからサービスレイヤーへの依存）
- `src/services/` → `src/cli/` （❌ サービスレイヤーからUIレイヤーへの依存）
- `src/storage/` → `src/cli/` （❌ データレイヤーからUIレイヤーへの依存）

### 循環依存の禁止

同一レイヤー内でのクラス間の循環依存は禁止。共通処理は `src/types/` に型定義として抽出するか、新しいサービスとして独立させる。

---

## スケーリング戦略

### 機能の追加方針

| 規模 | 対応方針 | 例 |
|------|---------|-----|
| 小規模（単一クラス追加） | 既存ディレクトリに追加 | `GitHubService.ts` を `services/` に追加 |
| 中規模（関連クラスが3個以上） | レイヤー内にサブディレクトリを作成 | `services/github/` を作成 |
| 大規模（独立したドメイン） | `modules/` として独立 | `modules/team-management/` |

### ファイルサイズ管理

- **目安**: 1ファイル300行以下を推奨
- **300〜500行**: リファクタリングを検討（責務の分割が可能か確認）
- **500行以上**: 分割を強く推奨

---

## 除外設定

### `.gitignore`

```
node_modules/
dist/
.task/config.json      # GitHubトークンを含む設定ファイル
*.log
.DS_Store
coverage/
```

**注意**: `.task/tasks.json` はGitで共有することでチーム間のタスク同期に使えるため、デフォルトでは除外しない。チームの方針に応じて各自で判断する。

### `.prettierignore` / `.eslintignore`

```
dist/
node_modules/
coverage/
.steering/
```
