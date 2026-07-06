---
name: git-dev-workflow
description: Git-Flow ベースのブランチ運用・コミット・CI・Pull Request のルールに従って開発を進めるためのスキル。ブランチを切る／コミットする／PR を作成する／リリースやホットフィックスを行う際に必ず参照する。C++ は clang-tidy + GoogleTest、Python は PyLint + pytest を前提とする。
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

### 作業チェックリスト

- [ ] 作業ブランチを `dev` から命名規則に沿って作成した
- [ ] コミットは細かく、push 前に squash した
- [ ] ローカルで静的解析（clang-tidy / PyLint）とテスト（GoogleTest / pytest）が Green
- [ ] PR 本文に上記 6 項目を記載した
- [ ] release / hotfix の場合は `main` と `dev` の両方に PR を出した
