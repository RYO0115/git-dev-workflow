# git-dev-workflow

Git-Flow をベースとしたチーム開発のワークフロー（ブランチ運用・コミット・CI・Pull Request）を定義したリポジトリ。ルールを Claude Code の **Skill** として整備し、開発作業時に自動で参照・適用できるようにする。

## 構成

| ファイル | 内容 |
| --- | --- |
| `git-rule-draft.md` | ルールの原案（日本語ドラフト）。ワークフローの一次情報。 |
| `SKILL.md` | `git-rule-draft.md` を Skill 形式にまとめたもの。C++ の空欄を clang-tidy / GoogleTest で補完し、共通部分を統合。 |
| `CLAUDE.md` | Claude Code 向けのリポジトリガイド。 |
| `README.md` | 本ファイル。 |

## 他プロジェクトへの導入

Claude Code の Skill は、対象ディレクトリ配下に `<skill-name>/SKILL.md` という構成で配置すると認識される。本リポジトリの `SKILL.md` を **`git-dev-workflow/SKILL.md`** として配置する。配置先は用途に応じて選ぶ。

| スコープ | 配置先 | 用途 |
| --- | --- | --- |
| プロジェクト単位 | `<project>/.claude/skills/git-dev-workflow/SKILL.md` | そのリポジトリを触る全員で共有（リポジトリにコミットする） |
| 個人（全プロジェクト） | `~/.claude/skills/git-dev-workflow/SKILL.md` | 自分のローカル環境の全プロジェクトで有効 |

### 方法1: 手動コピー

```bash
# 導入先プロジェクトのルートで実行
mkdir -p .claude/skills/git-dev-workflow
curl -fsSL https://raw.githubusercontent.com/RYO0115/git-dev-workflow/main/SKILL.md \
  -o .claude/skills/git-dev-workflow/SKILL.md
```

ローカルにクローン済みの場合は `cp path/to/git-dev-workflow/SKILL.md .claude/skills/git-dev-workflow/SKILL.md` でよい。

### 方法2: git subtree（更新を追従したい場合）

本リポジトリを upstream として取り込み、ルール更新を `git subtree pull` で反映できる。

```bash
# 導入先プロジェクトのルートで実行
git subtree add --prefix .claude/skills/git-dev-workflow \
  https://github.com/RYO0115/git-dev-workflow.git main --squash

# 以降、ルール更新を取り込むとき
git subtree pull --prefix .claude/skills/git-dev-workflow \
  https://github.com/RYO0115/git-dev-workflow.git main --squash
```

> subtree ではリポジトリ全体（`README.md` 等も含む）が取り込まれる。取り込み先ディレクトリ直下に `SKILL.md` が存在するため Skill として認識されるが、`git-rule-draft.md` など他ファイルも同梱される点に留意する。

### 方法3: git submodule（独立したリポジトリとして紐付けたい場合）

本リポジトリを導入先の一部ディレクトリにサブモジュールとして紐付ける。履歴は混ざらず、参照している commit を明示的に管理できる。

```bash
# 導入先プロジェクトのルートで実行
git submodule add https://github.com/RYO0115/git-dev-workflow.git \
  .claude/skills/git-dev-workflow
git commit -m "Add git-dev-workflow skill as submodule"

# 以降、ルール更新を取り込むとき
git submodule update --remote .claude/skills/git-dev-workflow
git add .claude/skills/git-dev-workflow
git commit -m "Update git-dev-workflow skill"
```

クローンした人がサブモジュールも取得できるよう、初回は以下を案内する。

```bash
git clone --recurse-submodules <導入先リポジトリ>
# クローン済みの場合
git submodule update --init --recursive
```

> submodule でもディレクトリ直下に `SKILL.md` が置かれるため Skill として認識される。subtree と違い導入先の履歴にファイル実体は含まれず、参照 commit（gitlink）のみが記録される。サブモジュールを取得していない環境では中身が空になり Skill が読み込まれない点に注意する。

### 導入後の確認

配置したら Claude Code を再起動し、`/` でスキル一覧に `git-dev-workflow` が表示されること、または frontmatter の `description` に沿った作業（ブランチ作成・PR 作成など）で自動的に参照されることを確認する。

## ワークフロー概要

### ブランチ

Git-Flow を採用する。

- **恒久ブランチ**: `main`（リリース済み）/ `dev`（統合・起点）。作成時のみで削除しない。
- **作業ブランチ**（いずれも `dev` から作成）:

  | 種別 | 命名規則 | PR 先 |
  | --- | --- | --- |
  | feature | `feature/機能名` / `feature/タスク番号_機能名` | `dev` |
  | release | `release/vX.Y.Z` | `main` と `dev` の両方 |
  | hotfix | `hotfix/issue番号_機能名` | `main` と `dev` の両方 |

- **バージョン**: `major.minor.build`。アーキテクチャ変更は major、機能追加・バグ修正は minor、細かい修正ごとに build をインクリメント。

### 並行開発（git worktree）

複数セッション・複数エージェントで同じリポジトリを同時に編集する場合は `git checkout` ではなく **git worktree** を使う（1 ブランチ = 1 worktree、置き場所はリポジトリの外）。

素の `git worktree add` は gitignore された作業リソース（submodule の中身・`.env`・仮想環境・ローカル設定）を持ってこないため、**リポジトリごとに worktree 作成スクリプトを用意し、それ経由でのみ作る**。またコミット対象の共有リソース（SQLite DB・生成データ等）は複数 worktree から同時に書き換えない。詳細と雛形スクリプトは `SKILL.md` を参照。

### コミット

細かくローカルコミットし、リモートへ push する際に squash してまとめる。


### CI（GitHub Actions）

PR 時に **静的解析** と **自動テスト** を実行し、失敗している場合はマージしない。

| 言語 | 静的解析 | テスト |
| --- | --- | --- |
| C++ | clang-tidy | GoogleTest |
| Python | PyLint | pytest |

### Pull Request

PR 本文に以下を記載する: ①追加機能/バグ概要、②根本原因（バグ修正時）、③主な変更内容、④確認項目一覧、⑤テスト、⑥ドキュメント。

## 使い方

詳細なルールと作業チェックリストは [`SKILL.md`](./SKILL.md) を参照。ブランチ作成・コミット・PR 作成の各操作を行う前に該当セクションに従うこと。
