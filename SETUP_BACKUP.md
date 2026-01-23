# 自動バックアップ設定ガイド

## 📋 概要

VA Base Portalのデータベース（Cloudflare D1）を毎日自動的にバックアップします。

---

## ✅ 実装完了

以下のファイルが作成されました：

1. **`.github/workflows/backup-database.yml`** - GitHub Actionsワークフロー
2. **`RESTORE_DATABASE.md`** - 復元手順書
3. **`backups/`** - バックアップ保存ディレクトリ
4. **`README.md`** - バックアップ機能の説明を追加

---

## 🔧 設定手順

### Step 1: Cloudflare API Token を取得

バックアップには Cloudflare API Token が必要です。

**既に取得済みの場合:**
- メール送信機能で使用した `CLOUDFLARE_API_TOKEN` を再利用できます

**新規取得する場合:**

1. Cloudflare Dashboard にログイン: https://dash.cloudflare.com
2. 右上のアカウントアイコン → 「My Profile」
3. 左メニューの「API Tokens」をクリック
4. 「Create Token」をクリック
5. テンプレート選択:
   - 「Edit Cloudflare Workers」を選択
   - または「Create Custom Token」
6. 権限設定:
   - **Account Resources**: `D1` - `Edit`
   - **Zone Resources**: 不要
7. 「Continue to summary」→「Create Token」
8. **トークンをコピー**（一度しか表示されません！）

---

### Step 2: GitHub Secrets に設定

1. GitHubリポジトリページ: https://github.com/YOUR_USERNAME/webapp
2. 「Settings」タブをクリック
3. 左メニューの「Secrets and variables」→「Actions」をクリック
4. 「New repository secret」をクリック
5. 設定内容:
   - **Name**: `CLOUDFLARE_API_TOKEN`
   - **Secret**: コピーしたAPIトークンを貼り付け
6. 「Add secret」をクリック

**確認:**
- Secrets一覧に `CLOUDFLARE_API_TOKEN` が表示されればOK

---

### Step 3: GitHubにコードをプッシュ

**方法A: GitHub Web UI（簡単）**

1. GitHubリポジトリページで「Add file」→「Upload files」
2. 以下のファイルをアップロード:
   - `.github/workflows/backup-database.yml`
   - `RESTORE_DATABASE.md`
   - `backups/.gitkeep`
3. Commit changes

**方法B: Git コマンド（ターミナル）**

```bash
cd /home/user/webapp

# リモートリポジトリを追加（まだの場合）
git remote add origin https://github.com/YOUR_USERNAME/webapp.git

# プッシュ
git push origin main
