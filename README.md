# aio-edsbyom-content

AEM Edge Delivery Services BYOM パイプラインのコンテンツリポジトリ。Markdown でコンテンツを管理し、GitHub Actions 経由で AIO オーケストレーターを呼び出してプレビュー・公開を行う。

## 役割

- コンテンツ編集者が Markdown でコンテンツを管理するリポジトリ
- `stage` ブランチへの push でプレビューを自動実行する
- `stage → main` の PR マージで公開を自動実行する
- `eds-preview` ブランチに AIO が生成した HTML を格納する
- `eds-prod` ブランチに公開済み HTML のミラーを保持する

## ブランチ構成

| ブランチ | 用途 | 管理者 |
|---------|------|--------|
| `main` | 公開済みソース (Markdown) | 編集者 (PR のみ) |
| `stage` | プレビュー対象ソース (Markdown) | 編集者 |
| `eds-preview` | AIO 生成 HTML + マニフェスト | AIO (自動) |
| `eds-prod` | 公開済み HTML ミラー (監査用) | AIO (自動) |

## コンテンツ編集フロー

```
feature/xxx ブランチ作成
  │
  ├─ Markdown 編集・コミット
  │
  ▼
stage へマージ
  │  GitHub Actions が自動実行 (preview.yml)
  │  AIO が HTML 生成 → eds-preview へコミット
  │  AEM Admin Preview API 呼び出し
  │  eds/preview ステータスチェックが success になる
  │
  ▼
stage → main へ PR 作成
  │  eds/preview の成功が必須条件 (ブランチ保護)
  │  1名以上のレビュー承認が必須条件
  │
  ▼
PR マージ (Squash merge 推奨)
  │  GitHub Actions が自動実行 (publish.yml)
  │  AIO がプレビューマニフェストから確認済み HTML を復元
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

テーブル記法でブロックコンポーネントを定義する。1行目のヘッダーがブロック名になる。

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

`---` (水平線) でセクションを区切る。各セクションは `<div>` でラップされる。

```markdown
# セクション1のコンテンツ

---

# セクション2のコンテンツ
```

### メタデータ

ページ末尾に `Metadata` ブロックを置く。

```markdown
| Metadata |
| -------- |
| Title | ページタイトル |
| Description | ページの説明文 |
| Template | テンプレート名 |
```

### サンプルページ

```markdown
# ページ見出し

本文テキスト。

| Hero |
| ---- |
| ヒーローコンテンツ |

---

| Columns |
| ------- | ----------- |
| 左カラム | 右カラム |

---

| Metadata |
| -------- |
| Title | ページタイトル |
| Description | ページ説明 |
```

## GitHub Actions ワークフロー

### preview.yml

- トリガー: `stage` ブランチへの push
- 環境: `stage` GitHub Environment
- 処理:
  1. 変更ファイル (`.md` / 画像等アセット) を検出
  2. `eds/preview` ステータスを pending に設定
  3. AIO オーケストレーターへ HMAC 署名付き POST
  4. 成功/失敗に応じて `eds/preview` ステータスを更新

### publish.yml

- トリガー: `main` ブランチへの PR マージ
- 環境: `production` GitHub Environment
- 前提条件: PR の `eds/preview` ステータスが success であること
- 処理:
  1. PR の `eds/preview` ステータスを確認
  2. AIO オーケストレーターへ HMAC 署名付き POST (publish モード)
  3. 成功/失敗に応じて `eds/publish` ステータスを更新

## ブランチ保護ルール

### `stage` ブランチ

- `eds/preview` ステータスチェック必須
- マージ前に最新状態との同期必須

### `main` ブランチ

- PR 必須 (直接 push 不可)
- レビュー承認 1名以上必須
- `eds/preview` ステータスチェック必須
- 保護設定のバイパス不可

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

`production` 環境は `main` ブランチからのみデプロイ可能に制限されている。

## セットアップ手順 (新規リポジトリ)

1. このリポジトリを fork またはテンプレートから作成する
2. `stage` と `main` のブランチ保護ルールを設定する ([参照](.github/branch-protection.md))
3. GitHub Environments (`stage`, `production`) を作成する
4. 各 Environment に `AIO_ACTION_URL` と `HMAC_SECRET` を Secret として登録する
5. AIO リポジトリをデプロイし、`AIO_ACTION_URL` を取得する
6. `eds-preview` と `eds-prod` のオーファンブランチを作成する

```bash
# eds-preview ブランチ作成
git checkout --orphan eds-preview
git rm -rf .
mkdir -p .eds/runs
echo '{"version":1,"pages":[]}' > .eds/manifest.json
git add .eds/manifest.json
git commit -m "chore: initialize eds-preview branch"
git push origin eds-preview

# eds-prod ブランチ作成
git checkout --orphan eds-prod
git rm -rf .
mkdir -p .eds/runs
echo '{"version":1,"pages":[]}' > .eds/manifest.json
git add .eds/manifest.json
git commit -m "chore: initialize eds-prod branch"
git push origin eds-prod

git checkout main
```
