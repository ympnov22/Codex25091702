# YouTube Video Availability Tracker - Specification

## 1. Overview
- Purpose: help track manually curated YouTube video URLs and detect when each video becomes inaccessible (deleted, private, region blocked, etc.).
- Scope: single administrator manages the dataset; public audience can browse the current status.

## 2. Roles & Access
- Administrator: single user authenticated via password-only login, can create/update/delete videos and trigger manual checks.
- Public User: read-only access to the published list without authentication.

## 3. Functional Requirements
- Video registration: admin submits a YouTube URL; system stores URL plus fetched metadata (title, channel, thumbnail) when available.
- Video editing: admin can modify stored metadata and optional notes.
- Manual availability check: admin triggers a batch endpoint that refreshes metadata and status for all or selected videos using YouTube Data API (and oEmbed fallback).
- Status tracking: each video carries a status (`active`, `gone`, `unknown`), `last_checked_at`, optional `removed_at`, and `removed_reason`.
- History logging: each check stores a row in `check_logs` for auditing (status, time, response detail).
- Public listing: unauthenticated view shows title, channel, thumbnail, current status, and removal timestamp/reason if gone.
- Search/filter: basic text search (title/channel) and status filters on admin view; public view starts with search, optional filters later.
- Authentication: admin login form compares submitted password against bcrypt hash stored as env var; sets secure HTTP-only session cookie; logout clears cookie.

## 4. Non-Functional Requirements
- Usability: minimal UI with straightforward table-based interactions.
- Security: no sensitive data beyond admin password hash; enforce HTTPS (Render-managed); session cookie `SameSite=Strict`, `Secure`.
- Performance: manual checks run synchronously when triggered; ensure individual request timeout/handling to avoid blocking entire batch.
- Reliability: manual checks should handle intermittent API failures with graceful degradation.
- Privacy: public page exposes only intended metadata.

## 5. UI Overview
- Admin dashboard:
  - Header with app name plus `Run Manual Check` button showing progress/toast on completion.
  - Table columns: Status icon, Thumbnail, Title, Channel, Last Checked, Removed At, URL, actions (edit/delete).
  - Filters and search bar; Add/Edit modal form (URL required, metadata editable, optional notes).
- Public view:
  - Header with app description.
  - List/grid entries: Thumbnail, Title (link to YouTube when active), Channel, Status badge, optional `Removed on` label and reason snippet when gone.
  - Simple search input; filters can be introduced later.

## 6. Data Model (initial)
- `videos`
  - `id` (uuid)
  - `url` (unique)
  - `title`
  - `channel`
  - `thumbnail_url`
  - `status` enum (`active`, `gone`, `unknown`)
  - `last_checked_at`
  - `removed_at`
  - `removed_reason`
  - `notes`
  - `created_at`
  - `updated_at`
- `check_logs`
  - `id` (uuid)
  - `video_id` (FK -> videos)
  - `status` (`active`, `gone`, `unknown`, `error`)
  - `checked_at`
  - `detail` (JSON/text for API response, HTTP status, etc.)

## 7. API Endpoints (JSON over HTTPS)
- `POST /api/auth/login` -> `{ password }` -> sets session cookie if valid.
- `POST /api/auth/logout` -> clears session.
- `GET /api/videos` -> public listing with optional `status`, `search`.
- `GET /api/admin/videos` -> admin listing (includes internal fields/notes).
- `POST /api/admin/videos` -> create video.
- `PATCH /api/admin/videos/:id` -> update.
- `DELETE /api/admin/videos/:id` -> delete.
- `POST /api/admin/videos/check` -> run manual check; optional body `{ ids?: string[] }`.
- `GET /api/admin/videos/:id/history` -> retrieve check logs (planned for later).

## 8. Manual Check Workflow
1. Admin presses `Run Manual Check`.
2. Backend batches URLs (configurable chunk size) and queries YouTube Data API `videos.list` (IDs extracted from URLs).
3. Interpret response: if item present and `privacyStatus` is `public/unlisted` -> `active`; if `private` -> `gone` with reason `Video is private`. If API returns empty list, fall back to oEmbed request to distinguish 404/410/401.
4. Update `videos` row with latest metadata/status timestamps; insert `check_logs` record containing raw status info.
5. Return summary (counts of active/gone/errors) to admin UI.

## 9. Technology Stack
- Frontend/Backend: Next.js 14 (App Router) with TypeScript.
- Styling: Tailwind CSS with optional Radix UI primitives.
- ORM: Prisma for schema migration and type-safe DB access.
- Database: PostgreSQL (Render managed instance).
- Session management: `iron-session` or custom signed cookie using Node crypto.
- Password hashing: bcrypt hash pre-generated and injected via environment variable.
- API calls: native `fetch`, with `googleapis` optional if helper library preferred.
- State management: React server components plus SWR/React Query in admin UI.
- Testing: Jest/Testing Library for units, Playwright for minimal e2e (future).

## 10. Deployment (Render Free Tier)
- Web Service: Next.js app deployed as Node service on Render (Region: Oregon/FRA). Build command `npm install && npm run build`; start command `npm run start -- --port $PORT`.
- Database: Render PostgreSQL (Free tier) for persistent storage; configure Prisma `DATABASE_URL`.
- Domain: use Render-provided subdomain initially; custom domain optional later.
- Manual checks: executed via admin-triggered HTTP POST; no separate worker required on free tier.
- Cold starts: free plan sleeps after inactivity; manual use tolerates wake-up delay.

## 11. Environment Variables
- `DATABASE_URL`
- `ADMIN_PASSWORD_HASH`
- `SESSION_SECRET`
- `YOUTUBE_API_KEY`
- Optional: `APP_BASE_URL`, `NODE_ENV`, `LOG_LEVEL`.

## 12. Future Enhancements (Backlog Candidates)
- Automated scheduled checks (Render cron or external scheduler).
- Notification integrations (email, Discord) when video status changes.
- CSV/JSON export for public list.
- Tagging/categorization of videos.
- Per-video check from admin table.
- Public API endpoint for consumption by other services.

---

## 日本語訳サマリー
1. 概要  
   手動で登録したYouTube動画URLの生存状況を監視し、消滅や非公開化を検知する。管理者は1名で、一般公開ビューから誰でも閲覧できる。

2. ロールとアクセス  
   管理者はパスワードのみでログインし、CRUDと手動チェックが可能。公開ユーザーは認証不要で閲覧のみ。

3. 機能要件  
   動画登録・編集、YouTube Data APIとoEmbedを使った手動チェック、`active/gone/unknown`のステータス管理、履歴ログ、公開リスト表示、検索・フィルタ、bcryptハッシュによるパスワード認証。

4. 非機能要件  
   シンプルなUI、HTTPSとセキュアクッキーによる保護、同期的手動チェックでもタイムアウト処理で安定性を確保、公開情報は必要最低限。

5. UI概要  
   管理画面はテーブルとモーダルで構成し、ボタンで手動チェック。公開画面はサムネイル、タイトル、チャンネル、ステータスを簡潔に表示。

6. データモデル  
   `videos` テーブルにURLやメタデータ、ステータス、タイムスタンプを保持。`check_logs` にチェック履歴と詳細を保存。

7. APIエンドポイント  
   認証ログイン/ログアウト、動画のCRUD、公開用リスト、手動チェック実行、履歴取得（将来）が揃う。

8. 手動チェック手順  
   管理者が実行 -> APIを呼び出し -> レスポンス解釈 -> DB更新＋ログ記録 -> サマリーを返却。

9. 技術スタック  
   Next.js 14 + TypeScript、Tailwind、Prisma、Render PostgreSQL、`iron-session`、bcrypt、SWR/React Query、Jest/Playwright。

10. Renderでのデプロイ  
    Webサービス1つで完結。無料Postgresを利用し、コールドスタートの遅延は許容。手動チェックはHTTP POSTで実施。

11. 環境変数  
    DB接続、管理者パスワードハッシュ、セッション秘密、YouTube APIキーなど。

12. 将来拡張  
    スケジュールチェック、通知、エクスポート、タグ、個別チェック、公開APIなどを検討余地あり。