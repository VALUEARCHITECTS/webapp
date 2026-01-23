# R2 バケットのバックアップについて

## 📋 概要

VA Base Portal では、経費申請の添付ファイル（領収書の画像）を **Cloudflare R2** に保存しています。

現在のバックアップ設定では：
- ✅ **データベース（D1）**: 完全バックアップ済み
- ⚠️ **R2 バケット（画像ファイル）**: メタデータのみバックアップ

---

## 🔍 現在のバックアップ内容

### ✅ バックアップされているもの

1. **データベース（D1）**
   - 全従業員データ
   - 全申請データ
   - 全承認履歴
   - 有給付与・使用履歴
   - **添付ファイルのURL情報**

2. **R2 メタデータ**
   - 添付ファイルのURLリスト
   - ファイル数
   - 申請との関連情報

### ⚠️ バックアップされていないもの

- **実際の画像ファイル**（JPG, PNG, PDFなど）
- Cloudflare R2 バケットの内容

---

## 🛡️ Cloudflare R2 の耐久性

### 自動保護機能

Cloudflare R2 には以下の自動保護機能があります：

- **99.999999999% (イレブンナイン) の耐久性**
- **複数データセンターへの自動レプリケーション**
- **データの自動冗長化**
- **ハードウェア障害からの自動回復**

つまり、**通常の運用では追加のバックアップは不要**です。

---

## 📊 リスク評価

### データ損失のリスク

| シナリオ | 発生確率 | 影響 |
|---------|---------|-----|
| Cloudflare のハードウェア障害 | 極めて低い | R2が自動回復 |
| Cloudflare のデータセンター障害 | 極めて低い | 自動レプリケーション |
| 誤操作による削除 | 低～中 | ⚠️ 復元不可 |
| アカウント削除 | 極めて低い | ⚠️ 復元不可 |

**最大のリスク：誤操作による削除**

---

## 🔧 追加の保護オプション

### オプション1: R2 バージョニング（推奨・無料）

ファイルの削除・上書き時に旧バージョンを保持します。

**設定方法：**
1. Cloudflare Dashboard → R2
2. `webapp-receipts` バケットを選択
3. Settings → Versioning → Enable

**メリット：**
- ✅ 無料
- ✅ 誤削除から保護
- ✅ ファイルの履歴を保持
- ✅ 簡単に復元可能

**デメリット：**
- ストレージ使用量が増加（古いバージョンも保存される）

---

### オプション2: S3 API でフルバックアップ（上級者向け）

AWS CLI を使って R2 バケット全体をダウンロードします。

**必要なもの：**
- R2 API キー（Access Key ID と Secret Access Key）
- AWS CLI のインストール

**手順：**

#### 1. R2 API キーを作成

```bash
# Cloudflare Dashboard → R2 → Manage R2 API Tokens → Create API Token
# Read & Write 権限を付与
Copy
2. GitHub Secrets に追加
R2_ACCESS_KEY_ID: your-access-key-id
R2_SECRET_ACCESS_KEY: your-secret-access-key
R2_ENDPOINT: https://your-account-id.r2.cloudflarestorage.com
3. ワークフローを更新
.github/workflows/backup-database.yml に以下を追加：

Copy- name: Install AWS CLI
  run: |
    sudo apt-get update
    sudo apt-get install -y awscli

- name: Backup R2 bucket (full)
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.R2_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.R2_SECRET_ACCESS_KEY }}
    AWS_ENDPOINT_URL: ${{ secrets.R2_ENDPOINT }}
  run: |
    BACKUP_DATE=$(TZ=Asia/Tokyo date +%Y-%m-%d)
    R2_BACKUP_DIR="backups/r2-full-${BACKUP_DATE}"
    mkdir -p "${R2_BACKUP_DIR}"
    
    # R2バケットの内容を全てダウンロード
    aws s3 sync s3://webapp-receipts "${R2_BACKUP_DIR}/" \
      --endpoint-url="${AWS_ENDPOINT_URL}"
    
    # 圧縮
    tar -czf "${R2_BACKUP_DIR}.tar.gz" -C backups "r2-full-${BACKUP_DATE}"
    rm -rf "${R2_BACKUP_DIR}"
    
    echo "R2 full backup completed"
メリット：

✅ 完全なバックアップ
✅ GitHub に保存
デメリット：

❌ API キーの管理が必要
❌ ファイルサイズが大きい場合、GitHub のストレージを圧迫
❌ 設定が複雑
🎯 推奨事項
現在の設定で十分なケース
以下の条件を満たす場合、現在のメタデータバックアップで十分です：

✅ Cloudflare R2 の耐久性を信頼している
✅ 誤削除のリスクが低い（管理者が少数）
✅ ファイル数が多くない（数百～数千件程度）
追加対策が推奨されるケース
以下の条件に当てはまる場合、R2 バージョニングを有効にすることを推奨します：

⚠️ 複数の管理者がファイルを操作する
⚠️ 誤削除のリスクがある
⚠️ ファイルの履歴を保持したい
フルバックアップが必要なケース
以下の条件に当てはまる場合、S3 API でのフルバックアップを検討してください：

⚠️ 法規制で物理的なバックアップが必要
⚠️ Cloudflare R2 以外にもコピーを保持したい
⚠️ 社内規定で複数の保存場所が必須
📊 現在のバックアップファイル構成
毎日のバックアップには以下が含まれます：

backups/
├── 2026-01-23.sql          # D1 データベースダンプ
├── 2026-01-23.sql.gz       # 圧縮版
└── r2-2026-01-23.tar.gz    # R2 メタデータ
    ├── file_list.json      # 完全なファイルリスト（申請IDと紐付き）
    ├── file_urls.txt       # ファイルURLのリスト
    └── README.md           # 説明ファイル
🔄 復元手順
データベースの復元
Copy# D1 データベースを復元
wrangler d1 import webapp-production --remote --file=backups/2026-01-23.sql
R2 ファイルの確認
Copy# バックアップされたファイルリストを確認
tar -xzf backups/r2-2026-01-23.tar.gz
cat r2-2026-01-23/file_urls.txt
注意事項
データベースを復元しても、実際の画像ファイルは復元されません
画像ファイルが失われた場合、申請詳細画面で「ファイルが見つかりません」エラーが表示されます
R2 バージョニングを有効にしていれば、Cloudflare Dashboard から復元可能です
🎊 まとめ
項目	状態	推奨対応
D1 データベース	✅ 完全バックアップ	なし
R2 メタデータ	✅ バックアップ済み	なし
R2 ファイル本体	⚠️ 未バックアップ	バージョニング有効化
推奨アクション：

今すぐ: R2 バージョニングを有効化（無料・簡単）
必要に応じて: S3 API でフルバックアップを設定
📚 関連ドキュメント
SETUP_BACKUP.md - バックアップの設定手順
RESTORE_DATABASE.md - データベース復元手順
Cloudflare R2 ドキュメント
R2 S3 API
