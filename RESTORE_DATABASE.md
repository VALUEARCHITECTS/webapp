# データベース復元手順書

## ⚠️ 重要な注意事項

**復元操作を行う前に必ずバックアップを取ってください！**

---

## 📦 バックアップの場所

すべてのバックアップは以下に保存されています：

- **GitHub リポジトリ**: `backups/` フォルダ
- **ファイル形式**: `YYYY-MM-DD.sql` と `YYYY-MM-DD.sql.gz`（圧縮版）

例：
backups/ ├── 2026-01-22.sql (本日のバックアップ) ├── 2026-01-22.sql.gz (圧縮版) ├── 2026-01-21.sql (昨日) ├── 2026-01-21.sql.gz └── ...


---

## 🔄 復元方法

### 方法1: 最新のバックアップから復元（推奨）

```bash
# 1. リポジトリをクローンまたはプル
git pull origin main

# 2. 最新のバックアップファイルを確認
ls -lh backups/ | tail -1

# 3. 復元実行（READ ONLY確認後）
# ⚠️ この操作は慎重に行ってください
wrangler d1 import webapp-production --remote --file=backups/2026-01-22.sql

# 4. 復元確認
wrangler d1 execute webapp-production --remote --command="SELECT COUNT(*) as employee_count FROM employees"
方法2: 特定の日付のバックアップから復元
Copy# 例: 2026年1月15日のバックアップから復元
wrangler d1 import webapp-production --remote --file=backups/2026-01-15.sql

# 復元確認
wrangler d1 execute webapp-production --remote --command="
SELECT 
  'employees' as table_name, COUNT(*) as count FROM employees
UNION ALL
SELECT 'applications', COUNT(*) FROM applications
UNION ALL
SELECT 'approval_history', COUNT(*) FROM approval_history
"
方法3: 圧縮ファイルから復元
Copy# 1. 圧縮ファイルを解凍
gunzip -k backups/2026-01-22.sql.gz

# 2. 復元実行
wrangler d1 import webapp-production --remote --file=backups/2026-01-22.sql
🔍 バックアップの内容確認（復元前）
復元する前に、バックアップファイルの内容を確認できます：

Copy# ファイルの最初の50行を表示
head -50 backups/2026-01-22.sql

# テーブル一覧を確認
grep "CREATE TABLE" backups/2026-01-22.sql

# 従業員数を確認（INSERT文の数）
grep "INSERT INTO employees" backups/2026-01-22.sql | wc -l
📊 バックアップ状況の確認
GitHub Web UI で確認
GitHubリポジトリページにアクセス
backups/ フォルダをクリック
ファイル一覧を確認
任意のファイルをクリックしてダウンロード可能
コマンドラインで確認
Copy# 全バックアップのリスト
ls -lh backups/

# 最新5件のバックアップ
ls -lt backups/*.sql | head -5

# バックアップの合計サイズ
du -sh backups/

# バックアップ数
ls backups/*.sql | wc -l
🚨 緊急復元手順
システムが完全にダウンした場合
新しいD1データベースを作成

Copywrangler d1 create webapp-production-restored
最新のバックアップから復元

Copywrangler d1 import webapp-production-restored --remote --file=backups/2026-01-22.sql
wrangler.jsonc を更新

Copy"d1_databases": [
  {
    "binding": "DB",
    "database_name": "webapp-production-restored",
    "database_id": "新しいデータベースID"
  }
]
再デプロイ

Copynpm run deploy
📅 バックアップスケジュール
頻度: 毎日
時刻: 深夜2:00 JST（UTC 17:00前日）
保存期間: 無期限（すべて保存）
形式: SQL（非圧縮）+ SQL.GZ（圧縮）
✅ 復元後の確認チェックリスト
復元後、以下を確認してください：

 従業員数が正しいか

Copywrangler d1 execute webapp-production --remote --command="SELECT COUNT(*) FROM employees"
 申請数が正しいか

Copywrangler d1 execute webapp-production --remote --command="SELECT COUNT(*) FROM applications"
 最新の申請日が正しいか

Copywrangler d1 execute webapp-production --remote --command="SELECT MAX(created_at) as latest_application FROM applications"
 ログインできるか

ブラウザで https://webapp-avq.pages.dev にアクセス
admin@value-arc.com でログイン
 申請一覧が表示されるか

ダッシュボードで申請一覧を確認
🔒 安全性について
バックアップ処理の安全性
✅ READ ONLY: バックアップは読み取り専用の操作です
✅ データ削除なし: wrangler d1 export はデータを読み取るだけです
✅ 自動化: GitHub Actions で自動実行されます
✅ 失敗時も安全: バックアップが失敗してもデータは影響を受けません
復元処理の注意事項
⚠️ 復元は慎重に: 復元操作は既存データを上書きします
⚠️ 事前確認: 復元前に必ず現在のデータをバックアップしてください
⚠️ テスト環境: 可能であれば、まずテスト環境で復元テストを行ってください
📞 トラブルシューティング
Q1: バックアップファイルが作成されない
確認項目:

GitHub Actions の実行ログを確認
CLOUDFLARE_API_TOKEN が設定されているか確認
Wranglerのバージョンを確認（最新版推奨）
Q2: 復元時にエラーが発生する
対処法:

バックアップファイルの内容を確認
Copyhead -50 backups/2026-01-22.sql
ファイルが破損していないか確認
Copyfile backups/2026-01-22.sql
Wranglerを最新版にアップデート
Copynpm install -g wrangler@latest
Q3: GitHub のストレージ容量が心配
対処法:

古いバックアップを削除（手動）
Copy# 90日より古いバックアップを削除
find backups/ -name "*.sql*" -mtime +90 -delete
または圧縮版のみ保持
Copy# 非圧縮版を削除（.gz は残す）
rm backups/*.sql
📚 関連ドキュメント
Wrangler D1 コマンド
Cloudflare D1 ドキュメント
GitHub Actions ドキュメント
📧 サポート
問題が発生した場合は、以下の情報を添えて報告してください：

エラーメッセージ
実行したコマンド
バックアップファイルの日付
GitHub Actions の実行ログ（該当する場合）

---
