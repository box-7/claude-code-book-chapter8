# Claude Code カスタマイズの仕組み

このプロジェクトでは、Markdownファイルを書くだけでClaude Codeの振る舞いをカスタマイズしています。
コードを一切書かずに `/setup-project` のようなコマンドや、専門的なスキルを定義できます。

## 全体の構造

```
/setup-project と入力する
    ↓ Claude Codeが読む
.claude/commands/setup-project.md   ← 「何をするか」の手順書
    ↓ 手順の中でSkillを呼ぶ
.claude/skills/*/SKILL.md           ← 各ドキュメントの「作り方ガイド＋テンプレート」
    ↓ 実行を許可するのが
.claude/settings.json               ← 権限設定
```

---

## 1. スラッシュコマンド → `.claude/commands/<name>.md`

ユーザーが `/setup-project` と入力すると、Claude Code は対応するファイルを読み込み、書かれた手順をそのまま実行します。

**ファイルパス**: `.claude/commands/setup-project.md`

```markdown
---
description: 初回セットアップ: 6つの永続ドキュメントを対話的に作成する
---

# 初回プロジェクトセットアップ

## 手順

### ステップ1: PRDの作成
1. prd-writingスキルをロード
2. docs/ideas/ の内容を元に docs/product-requirements.md を作成
...
```

**ポイント**:
- ファイル名がそのままコマンド名になる（`setup-project.md` → `/setup-project`）
- `description` フロントマターがコマンドの説明として表示される
- 手順はMarkdownで自由に記述できる

---

## 2. スキル → `.claude/skills/<name>/SKILL.md`

コマンドの手順の中で `Skill('prd-writing')` のように呼び出すと、Claude Code は該当スキルのファイル群を読み込みます。
スキルはガイド＋テンプレートのMarkdownファイルの集合です。

**ディレクトリ構造の例**:
```
.claude/skills/prd-writing/
├── SKILL.md       ← スキルのエントリーポイント（手順・ガイド）
└── template.md    ← PRDのテンプレート
```

**SKILL.md の例**:
```markdown
---
name: prd-writing
description: PRDを作成するための詳細ガイドとテンプレート。PRD作成時にのみ使用。
allowed-tools: Read, Write
---

# PRD作成スキル

## 手順
1. docs/ideas/initial-requirements.md を読む
2. テンプレート (./template.md) に従ってPRDを生成する
3. レビューして改善する
...
```

**ポイント**:
- `allowed-tools` でスキルが使えるツールを制限できる
- `template.md` など複数ファイルに分割して管理できる
- スキルはコマンドから呼び出す他、ユーザーが直接呼び出すこともできる

---

## 3. 権限設定 → `.claude/settings.json`

スキルの呼び出しを毎回確認なしで実行できるよう、許可リストに登録します。

**ファイルパス**: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Skill(prd-writing)",
      "Skill(functional-design)",
      "Skill(architecture-design)",
      "Skill(repository-structure)",
      "Skill(development-guidelines)",
      "Skill(glossary-creation)"
    ],
    "deny": [],
    "ask": []
  },
  "hooks": {}
}
```

**ポイント**:
- `allow` に追加したスキル・ツールはユーザー確認なしで実行される
- `deny` に追加したものは常に拒否される
- `ask` に追加したものは毎回確認が求められる（デフォルト動作）

---

## 4. 常時読み込まれる指示書 → `CLAUDE.md`

プロジェクトルートの `CLAUDE.md` は、会話のたびに自動で読み込まれるプロジェクト全体の指示書です。
技術スタック、開発プロセスのルール、ディレクトリ構造など「常に守ってほしいこと」を書きます。

```
CLAUDE.md                          ← 常時読み込まれる（プロジェクト全体の指示）
.claude/commands/<name>.md         ← /コマンド名 で呼び出せる指示
.claude/skills/<name>/SKILL.md     ← Skill() で呼び出せる専門指示
```

---

## このプロジェクトで定義されているコマンド・スキル一覧

### コマンド（`.claude/commands/`）

| コマンド | 説明 |
|---------|------|
| `/setup-project` | 6つの永続ドキュメントを対話的に作成する初回セットアップ |
| `/add-feature` | 新機能を既存パターンに従って実装する |
| `/review-docs` | ドキュメントの詳細レビューをサブエージェントで実行する |

### スキル（`.claude/skills/`）

| スキル名 | 用途 |
|---------|------|
| `prd-writing` | プロダクト要求定義書（PRD）の作成 |
| `functional-design` | 機能設計書の作成 |
| `architecture-design` | アーキテクチャ設計書の作成 |
| `repository-structure` | リポジトリ構造定義書の作成 |
| `development-guidelines` | 開発ガイドラインの作成 |
| `glossary-creation` | 用語集の作成 |
| `steering` | 作業計画・実装・振り返りのステアリングファイル管理 |
