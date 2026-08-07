---
name: git-dev-workflow
description: Git-Flow ベースのブランチ運用・コミット・CI・Pull Request のルールに従って開発を進めるためのスキル。ブランチを切る／コミットする／PR を作成する／リリースやホットフィックスを行う際に必ず参照する。複数セッション・複数エージェントで同じリポジトリを同時に編集するための git worktree 運用（作成スクリプト・共有リソースの排他）も含む。C++ は clang-tidy + GoogleTest、Python は PyLint + pytest を前提とする。
---

# Git 開発ワークフロー

Git-Flow を採用したチーム開発のルール。ブランチ作成・コミット・CI・PR 作成の各操作を行う前に、該当セクションを参照して従うこと。

## ブランチ運用

### 恒久ブランチ

| ブランチ | 役割 | 作成 | 削除 |
| --- | --- | --- | --- |
| `main` | リリース済みコード | リポジトリ作成時のみ | しない |
| `dev` | 統合・開発の起点 | リポジトリ作成時のみ | しない |

### 作業ブランチ

いずれも **`dev` から作成する**。命名規則と、完了後のマージ先（PR 先）は以下の通り。

| 種別 | 命名規則 | 用途 | 作業完了後の PR 先 |
| --- | --- | --- | --- |
| feature | `feature/機能名`<br>タスク番号があれば `feature/タスク番号_機能名` | 機能追加。1 機能につき 1 ブランチ | `dev` へ 1 本 |
| release | `release/vX.Y.Z` | リリース前の最終テスト | **`main` と `dev` の両方**へ 1 本ずつ |
| hotfix | `hotfix/issue番号_機能名` | issue 等で報告されたバグ修正 | **`main` と `dev` の両方**へ 1 本ずつ |

> **共通ルール（release / hotfix）**: 立ち位置は同じ。テスト・修正の完了後に `main` へ PR を出し、さらに `dev` へも PR を出して両ブランチへ反映する。`main` にだけ入れて `dev` へ戻し忘れることがないよう必ず 2 本作成する。

### 並行開発（git worktree）

複数のセッション（複数の Claude Code、複数のターミナル、複数の担当者）が**同時に別ブランチを編集する**場合、`git checkout` でブランチを切り替える運用は使えない。一方のチェックアウトが他方の作業ディレクトリを書き換えてしまうため。**git worktree** を使い、1 リポジトリに対して作業ディレクトリを複数持つ。

#### 原則

- **1 ブランチ = 1 worktree**。同じブランチを 2 つの worktree でチェックアウトすることは git が拒否する。
- worktree は**リポジトリの外**に置く。親ディレクトリ配下に `<repo>-worktrees/<branch のスラッシュを - に置換>/` を切るのが既定。リポジトリ内に置くと、テストランナーやパッケージ探索（例: pytest の `testpaths` / `pythonpath`、CMake の再帰探索）が worktree 側のコピーまで拾って原因の分かりにくい失敗をする。
- ブランチの命名規則・分岐元は[作業ブランチ](#作業ブランチ)と同じ。worktree にしたからといって `dev` 起点は変わらない。

#### 素の `git worktree add` を直接叩かない

`git worktree add` が持ってくるのは **git の追跡対象だけ**。gitignore されている作業用リソースは新しい worktree に存在せず、実行時に初めて失敗する。典型的には以下。

| 欠けるもの | 症状 |
| --- | --- |
| submodule の中身 | 空ディレクトリのまま。参照先のツール・設定が読み込まれない |
| `.env` 等の秘密情報 | API キー未設定でランタイムエラー |
| 仮想環境 / ビルドディレクトリ（`.venv`, `build/`） | 依存が解決できない・ビルドが通らない |
| エディタ / エージェントのローカル設定 | 権限プロンプトや設定が初期状態に戻る |

そのため、**リポジトリごとに worktree 作成スクリプトを用意し、worktree はそれ経由でのみ作る**。プロジェクトの `CLAUDE.md` にもそのスクリプトを使う旨を明記する（スキルは常時読み込まれるとは限らないため）。

スクリプトが担う処理は次の 5 つ。

1. `git fetch origin --prune`
2. `git worktree add` — ローカルブランチが既にあればそれを、`origin/<branch>` があれば追跡ブランチとして、どちらも無ければベースブランチ（既定 `dev`）から新規作成
3. `git submodule update --init --recursive`（入れ子 submodule があるので `--recursive` は必須）
4. gitignore された作業ファイルを本体（main worktree）からコピー
5. 依存導入（Python: `uv sync` / C++: CMake の configure）

雛形（Python + uv の例。`.env` やコピー対象はプロジェクトに合わせて差し替える）:

```bash
#!/usr/bin/env bash
set -euo pipefail
BRANCH="${1:?usage: $0 <branch> [base-branch]}"; BASE="${2:-dev}"

MAIN_ROOT="$(dirname "$(git rev-parse --path-format=absolute --git-common-dir)")"
WORKTREE_DIR="$(dirname "$MAIN_ROOT")/$(basename "$MAIN_ROOT")-worktrees/${BRANCH//\//-}"
[[ -e "$WORKTREE_DIR" ]] && { echo "error: $WORKTREE_DIR は既に存在します" >&2; exit 1; }

git -C "$MAIN_ROOT" fetch origin --prune
if git -C "$MAIN_ROOT" show-ref --verify --quiet "refs/heads/${BRANCH}"; then
  git -C "$MAIN_ROOT" worktree add "$WORKTREE_DIR" "$BRANCH"
elif git -C "$MAIN_ROOT" show-ref --verify --quiet "refs/remotes/origin/${BRANCH}"; then
  git -C "$MAIN_ROOT" worktree add --track -b "$BRANCH" "$WORKTREE_DIR" "origin/${BRANCH}"
else
  git -C "$MAIN_ROOT" show-ref --verify --quiet "refs/remotes/origin/${BASE}" \
    && BASE_REF="origin/${BASE}" || BASE_REF="${BASE}"
  git -C "$MAIN_ROOT" worktree add -b "$BRANCH" "$WORKTREE_DIR" "$BASE_REF"
fi

git -C "$WORKTREE_DIR" submodule update --init --recursive
for f in .env .claude/settings.local.json; do
  [[ -f "${MAIN_ROOT}/${f}" ]] || continue
  mkdir -p "$(dirname "${WORKTREE_DIR}/${f}")" && cp "${MAIN_ROOT}/${f}" "${WORKTREE_DIR}/${f}"
done
(cd "$WORKTREE_DIR" && uv sync)
```

#### 片付け

```bash
git worktree remove <path>   # 通常の削除
git worktree list            # 残っている worktree の確認
git worktree prune           # ディレクトリを手で消してしまった場合の後始末
```

マージ済みブランチの削除は通常どおり `git branch -d`。

#### コミット対象の共有リソースは排他する

**リポジトリにコミットされていて、かつプログラムが書き換えるファイル**（SQLite DB、スナップショット、生成済みデータセット、ロックファイル等）は、並行開発における最大の事故源。バイナリなら git はマージできず、テキストでも自動生成の差分は高確率でコンフリクトする。

- そうしたファイルを**書き換える処理を、2 つ以上の worktree で同時に走らせない**。
- プロジェクトの `CLAUDE.md` に、対象ファイル名と「同時に触らない」ルールを明記する。
- 並行作業するなら、**書き換えを伴うブランチは常に 1 本だけ**にし、他の worktree は読み取りのみの作業（ロジック実装・テスト・ドキュメント）に限定する。
- 作業開始前に `git worktree list` で他の worktree の存在を確認する。

#### その他の注意

- **仮想環境・ビルドディレクトリは worktree ごとに独立**。本体で依存を追加しても他の worktree には反映されない。各 worktree で `uv sync` / re-configure し直す。
- **秘密情報はコピーであって共有ではない**。本体の `.env` を更新したら、稼働中の worktree にも反映する。
- **submodule のコミットがブランチ間で違う場合**、チェックアウト後に `git submodule update --init --recursive` が要る。

### バージョン / タグ

#### バージョン番号

`major.minor.build`（例: `1.0.0`）で表記する。

- **major**: アーキテクチャが変わるような大規模変更でインクリメント
- **minor**: 機能追加・バグ修正レベルでインクリメント
- **build**: 細かい修正が入るたびにインクリメント

#### バージョンの管理場所（Single Source of Truth）

バージョン番号は言語ごとに以下のファイルで一元管理し、PR で変更内容に応じてインクリメントする。CI ではこのファイルの差分を検知し、バージョン更新漏れをチェックする（[Release 作業自動化](#release-作業自動化)の起点にもなる）。

| 言語 | 管理ファイル | 反映方法 |
| --- | --- | --- |
| C++ | `CMakeLists.txt`（`project(<name> VERSION X.Y.Z)`） | `version.h.in` を `configure_file()` で `version.h` に生成し、ソースから参照する |
| Python | `pyproject.toml`（`[project] version = "X.Y.Z"`） | パッケージから `importlib.metadata.version()` 等で参照する |

- **C++**: `version.h.in` にはプレースホルダ（例: `#define PROJECT_VERSION "@PROJECT_VERSION@"`）を記述し、CMake の `configure_file(version.h.in version.h)` でビルド時に実際の値へ展開する。生成物 `version.h` は `.gitignore` 対象とし、`CMakeLists.txt` と `version.h.in` のみをコミットする。
- **Python**: `pyproject.toml` の `version` を唯一の定義とし、コード内へバージョン文字列を直書きしない。

#### タグ付け

リリース時のタグ付けは手動では行わず、後述の [Release 作業自動化](#release-作業自動化)で `main` マージ後に `vX.Y.Z` タグを自動付与する。タグ名はバージョン番号に接頭辞 `v` を付けた形式とする。


## コミット

- コミットは**なるべく細かく**行い、push せずローカルで保持する。
- リモートへ push する際に **squash してコミットをまとめる**。開発履歴（build version ごとの実装内容）は PR 本文に残すため、リモート上のコミットは整理された状態にする。

## Github Action

### Release 作業自動化

`main` への Pull Request（release / hotfix 由来）が **merge された**ことを起点に、リリース作業を自動化する workflow を走らせる。手動でのタグ付け・リリース作成は行わない。

- **トリガー**: `on: pull_request` の `types: [closed]` で発火させ、`if: github.event.pull_request.merged == true && github.base_ref == 'main'` で「`main` へ実際に merge された PR」に限定する。
- **処理内容**:
  1. バージョン管理ファイル（C++: `CMakeLists.txt` / Python: `pyproject.toml`）からバージョン番号を取得する。
  2. `vX.Y.Z` タグを作成して push する（既存タグと重複する場合はエラーとし、バージョン更新漏れを検知する）。
  3. 当該タグで GitHub Release を作成する（リリースノートには PR 本文の内容を反映するとよい）。
- **前提設定**: リポジトリ設定で GitHub Actions に対しコンテンツ書き込み権限（`permissions: contents: write`）を付与し、PR merge で workflow が自動起動する状態にしておくこと。

### CI

Pull Request 時に以下を必ず走らせる。目的は「作り壊しの防止（リグレッション）」と「コード品質の担保（静的解析）」。

#### 言語共通の方針

PR トリガーで **静的解析** と **自動テスト** の両ジョブを実行し、どちらかが失敗している場合はマージしない。ローカルでも同じチェックを通してから PR を出す。

#### 言語別ツール

| 言語 | 静的解析 | テスト |
| --- | --- | --- |
| C++ | **clang-tidy** | **GoogleTest** |
| Python | **PyLint** | **pytest** |

##### C++

- 静的解析: `clang-tidy` を使用する。`compile_commands.json`（CMake の `CMAKE_EXPORT_COMPILE_COMMANDS=ON` で生成）を参照し、リポジトリ直下の `.clang-tidy` に有効化するチェックを定義する。
- テスト: `GoogleTest` を使用する。CMake（`enable_testing()` + `gtest_discover_tests`）で登録し、`ctest` から実行できるようにする。

##### Python

- 静的解析: `PyLint` を使用する。
- テスト: `pytest` を使用する。

### Pull Request

レビュアーが変更内容とその経緯を把握できるよう、PR 本文に以下を記載する。

1. **追加機能概要 / バグ概要**
   - なぜこの変更を行ったのかの背景と、大まかな変更内容。
   - 技術的詳細には触れず、中学生でもわかる噛み砕いた文章で説明する。
2. **根本原因**（バグ修正の場合）
   - なぜこのバグが発生したのかの根本原因。技術的な内容もここに記載してよい。
3. **主な変更内容**
   - 変更の詳細・技術的内容。開発中に build version が変わった場合は、各 build version ごとにどのような実装を行ったかを記載し、開発履歴を追えるようにする。
   - **UI に変更が入る場合は画面キャプチャを必ず貼る**（詳細は[画面キャプチャ](#画面キャプチャui-変更時は必須)）。
4. **確認項目一覧**
   - 実装内容に応じたテストの作成・実行計画を箇条書きで記載する。
   - 新規機能: 正常系・異常系それぞれにテストを実装する。
   - バグ修正: 再発防止のためのテストを実装し、動作を確認する。
5. **テスト**
   - 追加・修正したテストとその内容。新規テストは「何がどうなることを確認するか」をリストで表現する。
   - PR 提出前に手元でテストを実行し、**全て Green** を確認する。
   - GitHub Actions でテストが失敗している場合は、原因を調査・対応してから再提出する。
6. **ドキュメント**
   - 機能追加・バグ修正に伴い変更したドキュメントを記載する。

### 画面キャプチャ（UI 変更時は必須）

画面に出るものを変えた PR には、**変更後の画面キャプチャを PR 本文に貼る**。対象は
新しい UI 要素の追加・レイアウト変更・表示文言や配色の変更・生成ページの見た目の変更など、
「レビュアーがコードを読んでも実際どう見えるか分からない」もの全般。

- **撮るタイミングは実装中**。動作確認でブラウザを開いた時点でキャプチャを保存しておく。
  後から撮り直すには環境を組み直す必要があり、「もう残っていない」で省略されやすい。
- **撮る内容**は「変更の主眼が写っている状態」。メニューやダイアログなら**開いた状態**、
  条件で見た目が変わるなら**代表的な分岐ごと**に1枚。レスポンシブ対応が主眼なら
  デスクトップ幅と実機幅の両方。
- **置き場所はリポジトリ内**（例 `docs/images/`）にコミットし、設計ドキュメントからも
  参照する。PR 本文には raw URL
  （`https://github.com/<owner>/<repo>/raw/<branch>/docs/images/<file>`）で貼る。
  外部のアップロード先に置くと、PR がマージされた後にリンクが切れて履歴として残らない。
- キャプチャには**説明文を1行添える**（何の画面で、どこを見てほしいか）。
- 画面に変更が無い PR（内部リファクタ・収集ロジック・CI 設定など）では不要。

### 作業チェックリスト

- [ ] 作業ブランチを `dev` から命名規則に沿って作成した
- [ ] 並行セッションで作業する場合、worktree をプロジェクトの作成スクリプト経由で用意した
- [ ] コミット対象の共有リソース（DB・生成データ等）を他の worktree と同時に書き換えていない
- [ ] コミットは細かく、push 前に squash した
- [ ] ローカルで静的解析（clang-tidy / PyLint）とテスト（GoogleTest / pytest）が Green
- [ ] PR 本文に上記 6 項目を記載した
- [ ] UI に変更がある場合、画面キャプチャをリポジトリにコミットして PR 本文に貼った
- [ ] release / hotfix の場合は `main` と `dev` の両方に PR を出した
