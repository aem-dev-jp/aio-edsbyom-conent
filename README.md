# aio-edsbyom-content

AEM Edge Delivery Services BYOM パイプラインのコンテンツリポジトリ。Markdown でコンテンツを管理し、GitHub Actions 経由で AIO オーケストレーターを呼び出してプレビュー・公開を行う。

## 役割

- コンテンツ編集者が Markdown でコンテンツを管理する
- `stage` ブランチへの push でプレビューを自動実行する
- `stage → main` の PR マージで公開を自動実行する
- `eds-preview` ブランチに AIO が生成した HTML とマニフェストを格納する
- `eds-prod` ブランチに公開済み HTML のミラーを保持する

## ブランチ構成

| ブランチ | 用途 | 管理者 |
|---------|------|--------|
| `main` | 公開済みソース (Markdown) | 編集者 (PR のみ) |
| `stage` | プレビュー対象ソース (Markdown) | 編集者 |
| `eds-preview` | AIO 生成 HTML + マニフェスト | AIO（自動） |
| `eds-prod` | 公開済み HTML ミラー（監査用） | AIO（自動） |

## コンテンツ編集フロー

```
feature/xxx ブランチ作成（任意）
  │
  ▼
stage へ push（または直接 stage で編集）
  │  preview.yml が自動起動
  │  merge-base(origin/main, HEAD) からの差分で変更 .md を検出
  │  AIO が HTML 生成 → eds-preview へコミット
  │  AEM Admin Preview API 呼び出し
  │  eds/preview ステータスチェックが success になる
  │
  ▼
stage → main へ PR 作成
  │  PR head SHA の eds/preview が success であることを確認
  │  ブランチ保護の "bypass rules" は期待動作（後述）
  │
  ▼
PR をマージ
  │  publish.yml が自動起動
  │  AIO がマニフェストから確認済み HTML を復元
  │  AEM Admin Preview → Publish API 呼び出し
  │  eds-prod へミラー
  │  eds/publish ステータスチェックが success になる
```

## ディレクトリ構成

```
content/
├── index.md              トップページ
├── docs/
│   └── example.md        ドキュメント例
├── .github/
│   ├── workflows/
│   │   ├── preview.yml   stage push 時のプレビューワークフロー
│   │   └── publish.yml   main マージ時の公開ワークフロー
│   └── branch-protection.md  ブランチ保護設定メモ
└── .gitignore
```

## Markdown 記法

### 基本

CommonMark + GFM (GitHub Flavored Markdown) が利用できる。

### EDS ブロック

テーブル記法でブロックコンポーネントを定義する。1 行目のヘッダーがブロック名になる。

```markdown
| Hero |
| ---- |
| ヒーローテキスト |
```

複数カラムのブロック:

```markdown
| Columns |
| ------- | ----------- |
| 左カラム | 右カラム |
```

### セクション区切り

`---`（水平線）でセクションを区切る。各セクションは `<div>` でラップされる。

### メタデータ

ページ末尾に `Metadata` ブロックを置く。

```markdown
| Metadata |
| -------- |
| Title | ページタイトル |
| Description | ページの説明文 |
```

## GitHub Actions ワークフロー

### preview.yml

- **トリガー**: `stage` ブランチへの push、または `workflow_dispatch`
- **環境**: `stage` GitHub Environment
- **変更ファイル検出**: `git merge-base origin/main HEAD` を基準に差分を取得
  - `HEAD~1..HEAD` ではなく merge-base を使うことで、複数コミットをまとめて push した場合も `.md` の変更を確実に捕捉できる
- **処理**:
  1. `eds/preview` ステータスを pending に設定
  2. 変更ファイル（`.md` / 画像等アセット）を JSON 配列に変換（`jq -sc` でコンパクト出力 ※GITHUB_OUTPUT 仕様要件）
  3. AIO オーケストレーターへ HMAC 署名付き POST
  4. 成功 / 失敗に応じて `eds/preview` ステータスを更新

### publish.yml

- **トリガー**: `main` ブランチへの PR マージ (`pull_request: types: [closed]`)
- **環境**: `production` GitHub Environment
- **前提条件**: PR の `eds/preview` ステータスが success であること
- **処理**:
  1. PR head SHA の `eds/preview` ステータスを確認（success でなければ fail）
  2. AIO オーケストレーターへ HMAC 署名付き POST（publish モード）
  3. 成功 / 失敗に応じて `eds/publish` ステータスを更新

## GitHub Environments

### `stage` 環境

| Secret | 説明 |
|--------|------|
| `AIO_ACTION_URL` | AIO オーケストレーター Web Action の URL |
| `HMAC_SECRET` | AIO と共有する HMAC 署名シークレット |

### `production` 環境

| Secret | 説明 |
|--------|------|
| `AIO_ACTION_URL` | AIO オーケストレーター Web Action の URL |
| `HMAC_SECRET` | AIO と共有する HMAC 署名シークレット |

**重要**: `production` 環境の Deployment branches and tags は **"All branches"（制限なし）** に設定してください。

Publish ワークフローは `pull_request: closed` イベントで起動するため、GitHub がデプロイを評価する ref が `refs/pull/N/merge` になります。`main` ブランチのみを許可する custom policy では「not allowed to deploy」エラーになります。

```bash
# GitHub API で設定する場合
gh api --method PUT repos/{owner}/{repo}/environments/production \
  --field deployment_branch_policy=null
```

## ブランチ保護ルールの補足

`eds/preview` / `eds/publish` ステータスは GitHub Actions の **Checks API ではなく Commit Statuses API**（`github.rest.repos.createCommitStatus`）で書き込まれています。そのため：

- GitHub のブランチ保護 UI では `eds/preview` が「Required status check」として認識されず「bypass rules」の確認が表示されます
- これは **仕様上の期待動作** です。Publish ワークフロー自体が独自に `eds/preview: success` を API で確認するため、セキュリティ上の問題はありません
- PR マージ時に「Confirm bypass rules and merge」が表示されたら、そのままマージしてください

## セットアップ手順（新規リポジトリ）

1. このディレクトリを新しい GitHub リポジトリとして公開する
2. `eds-preview` と `eds-prod` のオーファンブランチを作成する

```bash
git checkout --orphan eds-preview
git rm -rf .
mkdir -p .eds/runs
echo '{"pages":{},"lastUpdated":null}' > .eds/manifest.json
git add .eds/manifest.json
git commit -m "chore: initialize eds-preview branch"
git push origin eds-preview

git checkout --orphan eds-prod
git rm -rf .
git commit --allow-empty -m "chore: initialize eds-prod branch"
git push origin eds-prod

git checkout main
```

3. GitHub Environments（`stage`, `production`）を作成する
4. 各 Environment に `AIO_ACTION_URL` と `HMAC_SECRET` を Secret として登録する
5. `production` 環境の Deployment branches を "All branches" に設定する
