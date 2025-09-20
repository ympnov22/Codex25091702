# 開発計画メモ

## 現状
- `docs/` に仕様ドキュメント (`youtube-video-availability-tracker-spec.md`) を格納。
- `web/` 配下に Next.js 14 + TypeScript + Tailwind の初期プロジェクトを生成済み。
- パッケージマネージャーは npm。開発用依存は `npm install` 済み。

## ローカル開発の始め方
1. `cd web`
2. `npm run dev`
3. ブラウザで `http://localhost:3000` を開き動作確認

## 今後の実装フェーズ（案）
1. Prisma 導入と DB スキーマ定義 (`videos`, `check_logs`)
2. 認証基盤 (`iron-session` + bcrypt 検証) と管理画面の保護
3. 管理画面 UI（一覧・検索・編集モーダル）
4. 公開用ビューと検索フォーム
5. 手動チェック API（YouTube Data API + oEmbed フォールバック）
6. チェック履歴保存とサマリー表示
7. デプロイ設定（Render）と環境変数の反映

## 決定事項・前提
- セッション管理は `iron-session` を採用。
- YouTube API Key / Render 環境はユーザー側で準備済み。
- DB は Render の PostgreSQL（Free Tier）を使用する想定。

## メモ
- Prisma 追加後は `web/prisma/schema.prisma` をルートに配置予定。
- テストはユニット（Jest/RTL）と将来的に e2e（Playwright）の導入を想定。
- 必要に応じて `docs/` 配下に追加設計ドキュメント（API 詳細、UI モックなど）を作成する。