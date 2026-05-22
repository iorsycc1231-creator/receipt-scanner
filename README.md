# Receipt Scan - レシートAI読取アプリ

Gemini AIでレシートを自動読取・経費精算CSV出力するWebアプリです。

## セットアップ手順

### 1. GitHubにアップロード

1. [github.com](https://github.com) でアカウント作成（無料）
2. 「New repository」でリポジトリ作成（名前例：`receipt-scanner`）
3. このフォルダの全ファイルをアップロード

### 2. Vercelにデプロイ

1. [vercel.com](https://vercel.com) にGitHubアカウントでログイン
2. 「New Project」→ GitHubのリポジトリを選択
3. そのまま「Deploy」ボタンを押す
4. 完了！ `https://あなたのアプリ名.vercel.app` のURLが発行されます

## 使い方

1. [aistudio.google.com](https://aistudio.google.com) でGemini APIキーを無料取得
2. アプリのAPIキー欄に貼り付け（デバイスに保存されます）
3. レシート画像をアップロード
4. 解析完了後、「CSVダウンロード」で経費精算用CSVを出力

## 費用

- Vercel：**無料**
- Gemini API：1日1,500リクエストまで**無料**（友達数人なら十分）
