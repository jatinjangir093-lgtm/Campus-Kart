# CampusKart

CampusKart is a student-focused marketplace built as a React/Vite client and a modular Express/Mongoose API.

## Local setup

Requirements: Node.js 22.12+ and a reachable MongoDB replica set. MongoDB Atlas works out of the box. For local MongoDB, start `mongod` with `--replSet rs0`, then initialize it once with `mongosh --eval "rs.initiate()"`. The verification review workflow uses database transactions.

1. Install dependencies:

   ```powershell
   cd .\client
   npm install
   cd ..\server
   npm install
   ```

2. Create local environment files:

   ```powershell
   Copy-Item .env.example .env
   cd ..\client
   Copy-Item .env.example .env
   ```

3. Configure `server\.env`. Set `MONGODB_URI`, a random `JWT_SECRET` of at least 32 characters, and `CLIENT_URL=http://localhost:5173`. Set a separate random `OTP_SECRET` as well. In local development, email delivery falls back to the explicitly labelled, loopback-only development mailbox when SMTP is not configured.

4. Add colleges to the registry before students register. For example, provide your own registry records (not production seed data) in PowerShell:

   ```powershell
   cd ..\server
   $env:COLLEGES_SEED_JSON='[{"name":"Example University","slug":"example-university","emailDomains":["students.example.edu"],"location":"Example City"}]'
   npm run seed:colleges
   Remove-Item Env:COLLEGES_SEED_JSON
   ```

5. Seed the default marketplace categories:

   ```powershell
   cd ..\server
   npm run seed:categories
   ```

   Category seeding is idempotent. Product image uploads require the Cloudinary credentials listed in `server\.env.example`; without them, listing creation returns an explicit storage configuration error.

6. Start the API and client in separate terminals:

   ```powershell
   cd .\server
   npm run dev
   ```

   ```powershell
   cd .\client
   npm run dev
   ```

   Open `http://localhost:5173`. The Vite proxy forwards `/api` requests to `http://localhost:5000`.

7. To bootstrap a trusted administrator, register an account, verify its college email, then run:

   ```powershell
   cd .\server
   npm run promote:super-admin -- verified-user@college.edu
   ```

   Public registration always creates a `student` role. Never expose the promotion script as an API endpoint.

## Environment configuration

`server\.env.example` lists the API variables. `MONGODB_URI`, `JWT_SECRET`, and `CLIENT_URL` are required. SMTP settings (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `EMAIL_FROM`) are required in production. Without SMTP in development, registration and resend responses explicitly say that delivery uses the development outbox; retrieve the code from the Verify Email screen or `POST /api/v1/auth/dev-mailbox` with `{ "email": "you@college.edu" }` on loopback. This route does not exist in production.

Student ID documents require a private S3-compatible bucket. Configure `AWS_REGION` and `S3_PRIVATE_BUCKET`, plus either the AWS SDK credential chain or `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. The bucket must block public access; the application does not set public ACLs and stores only opaque object keys. Set `S3_ENDPOINT` and `S3_FORCE_PATH_STYLE=true` when using a local S3-compatible service such as MinIO. Authorized reviewers receive a presigned document link valid for 60 seconds.

For production, also set the client’s `VITE_API_URL` to the HTTPS API base path (for example `https://api.example.com/api/v1`), set `CLIENT_URL` to the exact HTTPS frontend origin, and use HTTPS so secure cross-site cookies can be set. Configure the S3 bucket’s encryption, lifecycle, backups, access policy, and retention in the storage provider.

## Phase 2 checks

```powershell
cd .\server
npm test
npm audit
cd ..\client
npm run build
npm audit
```

## Phase 7 controlled beta operations

Phase 7 adds a server-enforced pilot layer. Students use `/app/beta`, `/app/feedback`, and `/app/safety`; admins use `/admin/beta`. The API lives beneath `/api/v1/beta`, including the scoped onboarding, feedback, support, ambassador, announcement, invitation, pilot configuration, flag, and aggregate analytics endpoints.

Start safely:

1. Create and verify an administrator, then open `/admin/beta`.
2. Keep the pilot disabled while configuring colleges, dates, registration limit, access mode, and limitations through `PATCH /api/v1/beta/admin/pilot/settings`.
3. Enable only the flags appropriate for the pilot. Flags are checked on the server; college overrides are supported by `PATCH /api/v1/beta/admin/features/:key`.
4. For invitation-only access, create time-limited, use-limited invitations with `POST /api/v1/beta/admin/invitations`. For approval mode, review `GET/PATCH /api/v1/beta/admin/pilot/registrations/:id`.
5. Publish a beta announcement and monitor feedback, support tickets, ambassador applications, and aggregate analytics. No private AI or message content is included in analytics.

Screenshot fields accept an HTTPS URL only. A private attachment store has not been added for Phase 7; configure one before collecting files from real students. Existing test-mode payment safeguards remain in force: no payment-provider approval, live deployment, or actual pilot onboarding has been verified from this repository. The operational handover, refund, incident, and backup procedures are in the Phase 6 deployment section above; do not introduce payment holding or escrow.

## Phase 6 production readiness and deployment

### Audit summary

The security review found one payment-accounting defect: a completed rental's deposit refund was written as a partial refund of the entire captured payment. Deposit refund state is now recorded independently (`depositRefundStatus`, `depositRefundAmountPaise`, and `depositRefundProviderId`); the overall payment remains `captured`, and general refund totals remain unchanged. Failed deposit refunds can be retried, and the rental transaction screen displays deposit-refund progress separately. Legacy in-flight deposit refunds are accepted by the settlement and failure handlers.

The read-only review found no other confirmed vulnerability in the inspected authentication/session, CSRF/CORS, authorization and college-scoping, private-document, upload-validation, payment-signature, Socket.IO, or AI tool-authorization paths. This is not a penetration test or a certification of production configuration. Provider approval, production secrets and infrastructure policies still require independent confirmation.

`GET /api/v1/health` is a liveness probe. `GET /api/v1/health/ready` pings MongoDB and returns only a generic `503` when the dependency is unavailable. The Render blueprint uses the readiness path. Production startup requires SMTP, private S3 storage, and an independent `OTP_SECRET` of at least 32 characters. Keep the environment keys in `server\.env.example` synchronized with the deployed service; the code uses `CLIENT_URL` (not `FRONTEND_URL`) and does not use a separate refresh-token secret or cookie-domain setting.

### Staging environment

Create separate Vercel and Render staging services and a separate MongoDB Atlas database/user. Use staging-only JWT/OTP secrets, email credentials, Cloudinary folder/account, AI key, and Razorpay **test-mode** credentials/webhook. Restrict staging access and outbound email as appropriate. Do not copy production student data or secrets to staging, and do not point staging at the production database. Seed only test colleges and accounts. Run the full tests and smoke checklist against staging before promoting a release.

### Deploy the frontend to Vercel

1. Import the repository and set the Vercel project root to `client`.
2. Use `npm ci`, `npm run build`, and `dist` as the install command, build command, and output directory. `client\vercel.json` rewrites SPA paths to `index.html`; `client\api\chat.js` is deployed as the `/api/chat` serverless function.
3. Configure `VITE_API_URL=https://<render-api-host>/api/v1`. This value is public configuration, never a secret.
4. Add `GEMINI_API_KEY` and `GEMINI_MODEL` in the Vercel project's server-side environment settings for Production, Preview, and Development as needed. Select a model ID currently supported by the Google Gemini API; do not add either value to a `VITE_` variable or frontend source.
5. Set the exact deployed frontend origin as Render's `CLIENT_URL`, including `https://`.
6. Add the custom domain and HTTPS in Vercel. Test direct navigation and refresh on `/login`, `/app`, and `/admin`, and smoke-test `/api/chat`.

### Deploy the API to Render

1. Create a persistent Render Web Service from the root `render.yaml` blueprint. It uses `server` as `rootDir`, `npm ci`, `npm start`, and `/api/v1/health/ready`; Socket.IO requires a persistent Node process and WebSocket upgrade support. Do not host this service as a serverless function.
2. Configure all prompted environment values from `server\.env.example`. Use unique production JWT and OTP secrets (each at least 32 random characters), the exact frontend origin in `CLIENT_URL`, and a MongoDB Atlas replica-set URI with TLS and a database user restricted to this app's database. Do not use a broad Atlas administrator account.
3. Configure SMTP and a private S3-compatible bucket with public access blocked. Apply encryption, least-privilege access, lifecycle/retention, and monitoring at the storage provider. Configure Cloudinary for public product images only.
4. Confirm Render's health check reaches `/api/v1/health/ready`, that HTTPS cookies work, and that `/api/socket.io` WebSocket upgrades reach the same persistent service. Keep logs free of credentials, OTP values, and private document contents.
5. Keep `RAZORPAY_MARKETPLACE_APPROVED=false` and `RAZORPAY_ALLOW_LIVE=false`. Configure test keys and the exact `/api/v1/payments/webhook` endpoint; subscribe to captured/failed payment and processed/failed refund events. Verify webhook delivery and refund handling with test transactions before considering any live configuration.
6. Configure `AI_API_KEY`, `AI_MODEL`, and optionally `AI_BASE_URL` only in the server environment. AI provider access is not part of readiness and must be independently smoke-tested.

### Production smoke checks

Before any controlled release, verify on the deployed staging stack:

- `/api/v1/health` and `/api/v1/health/ready`; readiness must fail safely when the database is unavailable.
- Registration, email delivery, OTP expiry/retry limits, login, refresh, logout, CSRF, secure cookies, and authorization for student/admin roles.
- Private student-ID uploads and authorized, short-lived reviewer access; confirm an ordinary student cannot access another student's document.
- Image upload and listing creation, college-scoped search, listing ownership, purchase and rental flows, overlap rejection, pickup/return OTP, and cancellation/refund paths.
- Razorpay test-mode signature and webhook verification, idempotency, duplicate event handling, full refunds, and deposit refunds including provider failure/retry. Confirm the overall rental payment remains captured when only its deposit is returned.
- Authenticated messaging, participant-only rooms, read state, notifications, AI provider failure behavior, college-scoped AI search, and admin permissions.
- Mobile and desktop layouts, browser console/network errors, deep links, and WebSocket reconnect behavior.

Automated tests use MongoDB in-memory replica sets and mocked provider/storage boundaries; they do not prove that Atlas, SMTP, S3, Cloudinary, Razorpay, the AI provider, Vercel, or Render are configured correctly.

### MongoDB backup, restore, and incident response

Use Atlas automated backups/PITR where available and ensure the plan matches the business's recovery-point and recovery-time requirements. As an additional encrypted off-site backup, run `mongodump` on a scheduled secure runner with a least-privilege account and protected output:

```powershell
mongodump --uri="$env:MONGODB_URI" --archive="campuskart-YYYYMMDD.archive.gz" --gzip
```

Recommended starting policy: daily backups, a 30-day rolling retention, and a longer monthly retention selected to meet legal and operational requirements. Restrict backup access, encrypt backups at rest and in transit, and test restores at least quarterly in an isolated database. For restore drills, restore to a fresh database and validate collections/indexes before any traffic points to it:

```powershell
mongorestore --uri="$env:RESTORE_MONGODB_URI" --archive="campuskart-YYYYMMDD.archive.gz" --gzip
```

Do not add `--drop` when restoring into a shared or production database without an approved recovery runbook and verified target. Private student documents require a separate S3 backup/retention plan. Record backup results and restore drills without including credentials.

For an incident: disable affected credentials/provider keys, restrict or suspend impacted accounts/services, preserve audit/provider logs, assess student/payment/document impact, notify the designated incident owner and required parties, restore only into a verified environment, rotate secrets, and document remediation. For application rollback, redeploy the last known-good Vercel and Render releases together and verify health, cookies, webhook delivery, and Socket.IO. No automated destructive database rollback is provided; take a backup and use a tested forward-fix or approved restore plan for schema/data changes.

### Launch gates

- [ ] Staging smoke checks and the test suite pass on the release candidate.
- [ ] Production Atlas backup/PITR and a restore drill are confirmed.
- [ ] SMTP, private student-document storage, image uploads, AI provider, webhook delivery, HTTPS, and WebSocket upgrades are verified with deployed configuration.
- [ ] Accessibility, responsive behavior, load/performance, and operational monitoring are reviewed.
- [ ] Payment provider has explicitly approved the marketplace model, seller onboarding/KYC, settlement, commission, refunds, and deposit handling; webhook and refund reconciliation are tested.

Do not enable live payments until all payment-provider and business/compliance approvals are confirmed. No cloud deployment, live provider verification, production-data migration, or restore drill has been performed by this project setup.

The integration tests use an in-memory MongoDB replica set and a mocked private object-storage boundary; they do not send real email or upload ID documents to a cloud bucket.

## Phase 3 marketplace

The authenticated marketplace lives at `/app/explore`; students can search only listings scoped to their verified college membership. Use `/app/create-listing` to publish a product and `/app/my-listings` to manage listings. Seed categories before using the listing form. Product photos are checked for JPG/PNG/WebP content and uploaded server-side to Cloudinary; they are public product photos, not private verification documents.

The Phase 3 integration suite mocks Cloudinary uploads while testing image validation and listing behavior. Messaging and listing reports remain deferred.

## Phase 4 transactions and payments

Students can reserve a sale listing, request rentals, pay through Razorpay Checkout, and confirm campus pickup/return with one-time codes. Rental ranges are represented as half-open UTC calendar dates `[startDate, endDate)` and protected by unique per-product/day inventory slots. Unpaid reservations expire automatically; the API worker releases them at least once per minute.

Razorpay is the only enabled payment provider. Add Razorpay **test-mode** `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET` plus a webhook secret to `server\.env` to test a real provider sandbox flow. Without them, the API reports that checkout is unavailable; there is no simulated payment-success path. Configure a Razorpay webhook for `/api/v1/payments/webhook` and subscribe to payment captured/failed and refund processed/failed events. Webhook signatures are verified against the exact raw request bytes.

The application deliberately blocks live Razorpay keys until `RAZORPAY_MARKETPLACE_APPROVED=true`. Do not enable live payments until Razorpay has approved this marketplace model and the business has completed seller onboarding/KYC, commission and settlement arrangements, refund handling, and rental-deposit terms. In non-production environments, live credentials additionally require `RAZORPAY_ALLOW_LIVE=true`. Razorpay standard orders do not implement escrow or seller split settlements; the application does not claim that sellers have been paid out. Rental deposits are collected in the test checkout total and can be returned by an administrator only after OTP-confirmed return. The admin transaction dashboard supports provider-evidenced full refunds and deposit refunds.

The marketplace commission is an integer basis-point rate configured by `PLATFORM_COMMISSION_BPS` (default 500; maximum 2000). Monetary values remain integer paise. Admin transaction pages are available at `/admin/transactions`; participants can open a dispute from a transaction detail screen. Disputed transactions pause handover until reviewed.

Phase 4 checks use MongoDB in-memory replica sets and a mocked Razorpay adapter; they validate provider signatures, amounts, idempotency, purchase concurrency, date-overlap locking, OTP handovers, refunds, and regressions. They do not charge a real card or prove provider marketplace approval.

### Phase 4 checks

```powershell
cd .\server
npm test
npm audit
cd ..\client
npm run build
npm audit
```

## Phase 5 messaging, CampusKart AI and administration

Student messages are stored in college-scoped conversations linked to a listing. Socket.IO uses `/api/socket.io` so the existing HTTP-only access cookie (scoped to `/api`) is sent by browsers, checks the active refresh session, and allows conversation rooms only after participant authorization. The Vite `/api` proxy enables websocket forwarding. REST remains the durable message path, while Socket.IO delivers new messages, read receipts, notifications, and same-college presence. Production ingress must forward websocket upgrades for `/api/socket.io`.

Relevant API routes:

- `GET/POST /api/v1/messages/conversations`
- `GET /api/v1/messages/conversations/:id` and `GET/POST /api/v1/messages/conversations/:id/messages`
- `PATCH /api/v1/messages/conversations/:id/read`
- `GET /api/v1/notifications/unread-count`, `PATCH /api/v1/notifications/read-all`, and existing list/read routes
- `GET/POST /api/v1/ai/conversations`, `GET/POST /api/v1/ai/conversations/:id/messages`, and `DELETE /api/v1/ai/conversations/:id`
- `GET /api/v1/ai/account-status` returns only the signed-in student's latest five purchase, sale, rental-booking, and rental-request summaries; it is authenticated and rate-limited.
- Admin tools: `/api/v1/admin/users`, `/listings`, `/reports`, `/disputes`, `/analytics`, `/policies`, and the existing college, verification, transaction, audit, and payment routes.

CampusKart AI uses controlled server-side tools. Product results and price filters come from the existing college-scoped product service; rental availability is checked through the existing rental availability service; support answers retrieve active FAQ/policy records. Listing drafts are reviewable only and are passed to the existing create-listing form without publishing. AI chat history is private to its owner and expires according to the MongoDB TTL setting.

For provider-generated natural language, configure `AI_API_KEY`, `AI_MODEL`, and optionally `AI_BASE_URL` in `server\.env`. The adapter uses an OpenAI-compatible Chat Completions endpoint and applies a timeout. No API key is required to exercise controlled product search, rental checks, policy retrieval, and listing drafts in development; that fallback is visibly labeled as development mode and does not claim to be real AI. In production, AI requests return an explicit unavailable response until the provider is configured. Provider keys are server-only.

### Floating Gemini chatbot

The floating **CampusKart AI** widget is available throughout the React app. Its Vercel Node function is `client\api\chat.js`, exposed at `POST /api/chat`, and uses the official `@google/genai` SDK. Configure `GEMINI_API_KEY` and `GEMINI_MODEL` only in Vercel server-side environment settings (or an ignored local `client\.env.local` for local Vercel CLI testing). The model value must be an ID supported by the Google Gemini API. Never use a `VITE_` prefix for either secret/config value. The older authenticated `/api/v1/ai` page and its `AI_API_KEY` settings are separate and unchanged.

The Vercel function has no access to live listings or account data. The widget directs live listing searches to Explore. For signed-in personal order/rental questions, it uses the authenticated, rate-limited `/api/v1/ai/account-status` API route, which queries records scoped to the current student and returns only item titles and transaction/payment statuses; verification status comes from that student's authenticated session profile. It never includes participant contact information, payment details, or OTPs. Account-specific messages are not sent to Gemini or persisted in browser chat history; if API data is unavailable, the student is directed to their dashboard. General workflow questions go to Gemini. The assistant does not confirm payments, generate OTPs, create transactions, or change handover/return state. General chat history is stored in the browser's local storage under a per-account key; clearing chat removes that browser history. The Vercel endpoint validates body size and shape, limits each runtime/IP bucket to 12 requests per minute, and returns explicit configuration/provider errors. The in-memory limit is per serverless runtime instance, not a globally coordinated quota.

#### Local setup and smoke test

From PowerShell:

```powershell
cd .\client
npm ci
@'
GEMINI_API_KEY=your_google_ai_studio_key
GEMINI_MODEL=use_a_current_supported_gemini_model_id
VITE_API_URL=http://localhost:5000/api/v1
'@ | Set-Content -Encoding ascii .env.local
npx vercel dev --listen 5173
```

Replace the example key/model values locally; `.env.local` is ignored and must not be committed. Sign in if testing authenticated marketplace pages; the chatbot itself does not read account data. Open `http://localhost:5173`, use the floating CampusKart AI button, and try an English and Hindi question. Also try a personal status question (it should direct you to the dashboard). To test the API directly, run `$body = @{ message = 'How do I rent an item?'; history = @() } | ConvertTo-Json` followed by `Invoke-RestMethod -Uri http://localhost:5173/api/chat -Method Post -ContentType 'application/json' -Body $body`. A `200` with a reply confirms the function and Gemini configuration; a `503` indicates missing/invalid server configuration or a temporarily unavailable provider. `npm run dev` alone serves the Vite UI but does not execute Vercel functions; use `vercel dev` for the integrated local test.

#### Vercel deployment

Set the project root to `client`, add `GEMINI_API_KEY` and `GEMINI_MODEL` under **Project Settings → Environment Variables** for each target environment, and deploy. Keep `VITE_API_URL` pointed at the existing API service; it is public configuration. Vercel discovers `api\chat.js` as `/api/chat` automatically. Deploy the updated `server` service as well so the authenticated `/api/v1/ai/account-status` route is available. Redeploy after changing environment variables, then verify the widget and `POST /api/chat` in a Preview deployment before Production. Do not put Gemini credentials in frontend settings, client source, committed env files, browser requests, or logs. For strict cross-instance rate enforcement at high scale, configure a shared rate-limit store or an edge/API gateway; the built-in function limiter is intentionally a bounded per-instance safeguard.

Default FAQ and policy records are inserted when the API starts and can be edited in `/admin/policies`; edits are preserved on later starts. Only admins/super-admins can manage students, colleges, analytics, transactions, and policies. Moderators can review student verifications, reports, and listing moderation. Admin actions are audit logged. Private student-ID documents remain accessible only through the existing authorized verification review flow.

Student routes include `/app/messages`, `/app/messages/:conversationId`, `/app/notifications`, and `/app/ai-assistant`. Admin sections include `/admin/users`, `/admin/colleges`, `/admin/listings`, `/admin/reports`, `/admin/analytics`, and `/admin/policies`.

Phase 5 tests use an in-memory MongoDB replica set; they do not call an external AI provider or send messages through an external service. Set `AI_CONVERSATION_RETENTION_DAYS` to a value from 1 to 3650 (default 90). `AI_TIMEOUT_MS` defaults to 12000 milliseconds.

```powershell
cd .\server
npm test
npm audit
cd ..\client
npm run build
npm audit
```
