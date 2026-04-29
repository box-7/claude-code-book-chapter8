# 開発ガイドライン (Development Guidelines)

## コーディング規約

### 命名規則

#### 変数・関数

```typescript
// ✅ 良い例
const taskList = await taskManager.listTasks();
function generateBranchName(taskId: string, title: string): string { }
const isGitRepository = await gitService.isGitRepository();

// ❌ 悪い例
const data = await mgr.list();
function gen(id: string, t: string): string { }
```

**原則**:
- 変数: `camelCase`、名詞または名詞句
- 関数: `camelCase`、動詞で始める
- 定数: `UPPER_SNAKE_CASE`（例: `MAX_TITLE_LENGTH = 200`）
- Boolean: `is`, `has`, `can`, `should` で始める

#### クラス・インターフェース・型

```typescript
// クラス: PascalCase + 役割サフィックス
class TaskManager { }
class GitService { }
class FileStorage { }

// インターフェース: I プレフィックス + PascalCase
interface IStorage { }

// 型エイリアス: PascalCase
type TaskStatus = 'open' | 'in_progress' | 'completed' | 'archived';
type TaskPriority = 'high' | 'medium' | 'low';
```

### コードフォーマット

- **インデント**: 2スペース
- **行の長さ**: 最大100文字
- **セミコロン**: あり
- **クォート**: シングルクォート
- **末尾カンマ**: あり（`es5`オプション）

フォーマットはPrettierで自動化するため、手動での調整は不要。

### コメント規約

コメントはコードを見ても分からない「なぜ」のみを書く。実装の説明コメントは書かない。

```typescript
// ✅ 良い例: 制約・理由を説明
// UUIDv4の先頭6文字は衝突確率が十分低く、CLIでの入力コストを下げるため短縮IDとして使用
const shortId = task.id.slice(0, 6);

// ✅ 良い例: 回避策の理由
// simple-gitはブランチ名に'/'が含まれていても作成できるが、日本語タイトルはslug化が必要
const branchName = generateBranchName(task.id, task.title);

// ❌ 悪い例: コードを言い直しているだけ
// ブランチ名を生成する
const branchName = generateBranchName(task.id, task.title);
```

**パブリックAPIのドキュメント**:

```typescript
/**
 * タスクを開始し、対応するGitブランチを作成・切り替える
 *
 * @param id - タスクID（先頭6文字以上の一致で検索）
 * @returns 更新されたタスク
 * @throws {NotFoundError} 一致するタスクが存在しない場合
 */
async startTask(id: string): Promise<Task> { }
```

### エラーハンドリング

**カスタムエラークラスを使用**:

```typescript
// src/types/errors.ts
export class AppError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public field: string) {
    super(message);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} が見つかりません (ID: ${id})`);
  }
}
```

**原則**:
- 予期されるエラー（入力不正・ID未発見）は適切なクラスで `throw`
- 予期しないエラーは握りつぶさず上位に伝播させる
- CLIレイヤーでのみユーザーへのメッセージとして表示する

```typescript
// ✅ 良い例: CLIレイヤーでまとめて処理
try {
  const task = await taskManager.startTask(id);
  formatter.displaySuccess(task);
} catch (error) {
  if (error instanceof NotFoundError) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
  throw error; // 予期しないエラーは上位へ
}

// ❌ 悪い例: サービスレイヤーでエラーを握りつぶす
async startTask(id: string): Promise<Task | null> {
  try {
    return await this.findAndStart(id);
  } catch {
    return null; // 情報が失われる
  }
}
```

### 非同期処理

`async/await` を使用する。Promiseチェーンは使わない。

```typescript
// ✅ 良い例
async function loadAndStart(id: string): Promise<Task> {
  const tasks = await storage.load();
  const task = tasks.find(t => t.id.startsWith(id));
  if (!task) throw new NotFoundError('タスク', id);
  return task;
}

// ❌ 悪い例
function loadAndStart(id: string): Promise<Task> {
  return storage.load()
    .then(tasks => tasks.find(t => t.id.startsWith(id)))
    .then(task => { if (!task) throw new Error(); return task!; });
}
```

---

## Git運用ルール

### ブランチ戦略

TaskCLI自体の開発では、**Git Flow** を採用する。

```
main（本番・リリース済み）
└── develop（開発統合）
    ├── feature/task-management    # 新機能
    ├── fix/branch-creation-error  # バグ修正
    └── refactor/storage-layer     # リファクタリング
```

**ルール**:
- `main` と `develop` への直接コミット禁止。必ずPRを経由する
- `feature/*` / `fix/*` は `develop` から分岐し、`develop` へマージ
- リリース時は `develop` → `main` へマージし、semverタグを付与

**TaskCLIが生成するブランチ**（ユーザープロジェクト用）:
- `task start <id>` で `feature/task-<id短縮形>-<slug>` を自動生成
- このブランチ命名はTaskCLIのロジックが管理する

### コミットメッセージ規約（Conventional Commits）

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Type**:
- `feat`: 新機能
- `fix`: バグ修正
- `refactor`: リファクタリング（機能変更なし）
- `test`: テスト追加・修正
- `docs`: ドキュメント
- `chore`: ビルド・ツール設定
- `perf`: パフォーマンス改善

**例**:
```
feat(git): task start時にブランチを自動作成する

task start <id> を実行したとき、feature/task-<id>-<slug> 形式の
Gitブランチを自動で作成・切り替えるようにした。

- GitService.generateBranchName() を追加
- 既存ブランチが存在する場合は切り替えのみ実施
- Gitリポジトリ外ではブランチ作成をスキップ

Closes #12
```

### プルリクエストプロセス

**作成前チェックリスト**:
- [ ] `npm run typecheck` がパスする
- [ ] `npm run lint` がパスする
- [ ] `npm test` が全件パスする
- [ ] 変更行数が300行以内（超える場合は分割を検討）

**PRテンプレート**（`.github/PULL_REQUEST_TEMPLATE.md`）:

```markdown
## 変更の種類
- [ ] 新機能 (feat)
- [ ] バグ修正 (fix)
- [ ] リファクタリング (refactor)
- [ ] ドキュメント (docs)

## 何を変更したか
[簡潔な説明]

## なぜ変更したか
[背景・理由]

## どのように変更したか
- [変更点1]
- [変更点2]

## テスト
- [ ] ユニットテスト追加
- [ ] 統合テスト確認
- [ ] 手動動作確認済み

## 関連Issue
Closes #[番号]

## レビューポイント
[特に見てほしい点があれば記載]
```

---

## テスト戦略

### テストピラミッド

```
       /\
      /E2E\       10%（Gitあり/なしシナリオ）
     /------\
    / 統合   \    20%（ファイルシステムを使ったCRUD）
   /----------\
  / ユニット   \  70%（サービス・バリデータ・フォーマッタ）
 /--------------\
```

**カバレッジ目標**:
- 全体: 80%以上
- `src/services/`: 90%以上（ビジネスロジックのコア）
- `src/validators/`: 90%以上（入力検証のコア）

### テストの書き方（Given-When-Then パターン）

```typescript
describe('TaskManager', () => {
  describe('startTask', () => {
    it('存在するタスクIDでブランチが作成される', async () => {
      // Given
      const storage = new MockFileStorage([mockTask]);
      const git = new MockGitService({ isRepo: true });
      const manager = new TaskManager(storage, git);

      // When
      const result = await manager.startTask('a1b2c3');

      // Then
      expect(result.status).toBe('in_progress');
      expect(result.branch).toBe('feature/task-a1b2c3-test-task');
      expect(git.createdBranch).toBe('feature/task-a1b2c3-test-task');
    });

    it('存在しないIDではNotFoundErrorがスローされる', async () => {
      // Given
      const storage = new MockFileStorage([]);
      const manager = new TaskManager(storage, new MockGitService());

      // When / Then
      await expect(manager.startTask('xxxxxx')).rejects.toThrow(NotFoundError);
    });
  });
});
```

**テスト命名規則**: 日本語で「〜の場合、〜になる」の形式を推奨。

### モック方針

- **ユニットテスト**: `IStorage` と `GitService` をモック化してサービスを単体テスト
- **統合テスト**: 実際のファイルシステム（一時ディレクトリ）を使用
- **E2Eテスト**: CLIコマンドを `child_process.spawnSync` で実行し、stdout/stderrを検証

```typescript
// ユニットテスト用モック例
const mockStorage: IStorage = {
  load: vi.fn().mockReturnValue({ version: '1.0.0', tasks: [mockTask] }),
  save: vi.fn(),
  backup: vi.fn(),
  exists: vi.fn().mockReturnValue(true),
  initialize: vi.fn(),
};
```

---

## コードレビュー基準

### レビューポイント

**機能性**:
- [ ] PRDの要件を満たしているか
- [ ] エッジケース（Gitリポジトリなし・IDが見つからない等）が考慮されているか
- [ ] エラーハンドリングが適切か

**可読性**:
- [ ] 命名が意図を明確に表しているか
- [ ] コメントが「なぜ」を説明しているか（「何をしているか」ではない）

**設計**:
- [ ] レイヤーの依存方向が正しいか（CLIがStorageに直接依存していないか等）
- [ ] 責務が単一か（1クラス・1関数が複数の責務を持っていないか）

**セキュリティ**:
- [ ] 入力検証が実装されているか
- [ ] GitHubトークン等の機密情報がハードコードされていないか
- [ ] ブランチ名生成でコマンドインジェクションの余地がないか

### レビューコメントの書き方

優先度を明示する:

```markdown
[必須] セキュリティ: ブランチ名がシェル文字列として渡されている可能性があります。
       simple-git の API 経由に変更してください。

[推奨] パフォーマンス: tasks.json を毎回全件読み込んでいます。
       start/done 等の書き込み操作前の1回のみ読み込む設計に統一してください。

[提案] 可読性: `isGitRepo` よりも `isGitRepository` の方が意図が明確です。

[質問] この `slice(0, 6)` は何を意図していますか？短縮IDでしょうか？
```

---

## 開発環境セットアップ

### 必要なツール

| ツール | バージョン | インストール方法 |
|--------|-----------|-----------------|
| Node.js | v24.11.0 | `nvm install 24` または devcontainer使用 |
| npm | 11.x | Node.jsに同梱 |
| Git | v2.20以上 | OS標準またはhomebrew |

### セットアップ手順

```bash
# 1. リポジトリのクローン
git clone https://github.com/your-org/task-cli.git
cd task-cli

# 2. 依存関係のインストール
npm install

# 3. ビルド確認
npm run build

# 4. テスト実行
npm test

# 5. 動作確認（ローカルインストール）
npm link
task --help
```

### 利用可能なnpm scripts

```json
{
  "scripts": {
    "build": "tsup",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "format": "prettier --write .",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "prepare": "husky"
  }
}
```

### CI/CD（GitHub Actions）

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: 'npm'
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

---

## 実装チェックリスト

実装・PR作成前に確認:

### コード品質
- [ ] 命名が明確で一貫している
- [ ] 関数・クラスが単一責務を持っている
- [ ] マジックナンバーが定数化されている
- [ ] 型注釈が適切に記載されている（`any` を使っていない）

### セキュリティ
- [ ] 入力バリデーションが `src/validators/` に実装されている
- [ ] GitHubトークン等がコード内にハードコードされていない
- [ ] ブランチ名生成はslug化関数を通じている

### テスト
- [ ] 新機能にユニットテストが書かれている
- [ ] 異常系（エラーケース）もテストされている
- [ ] `npm test` が全件パスする

### ツール
- [ ] `npm run typecheck` がエラーなし
- [ ] `npm run lint` がエラーなし
- [ ] `npm run format` 適用済み
