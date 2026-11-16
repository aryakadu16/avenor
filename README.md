# AVENOR — Intelligent communication. Effortless control.

## Status (read first)
| Area | State |
|---|---|
| Frontend (`frontend/`) | Approved design. **Sign-in/registration/logout/session are connected to Supabase Auth** (see Authentication). Emails, charts, approvals, follow-ups, notifications and settings still use the generated demo data and are **not yet connected to the API**. |
| Express API (`server/`) | Implemented. Unit/integration tests run **without** Supabase/OpenAI (see Tests). |
| Database (`database/migrations/001_schema.sql`) | Written, **never executed** against a real Supabase project. |
| Supabase Auth | Frontend + API integrated. Tested only against a **mock** Supabase, **not a real project**. |
| OpenAI | Server calls the Chat Completions API with validated JSON output. **Never run with a real key.** |
| Email sending | Not implemented. Approved replies are stored as `awaiting_sending_configuration`. |
| Demo seeding, Realtime client, browser push, creator/security dashboard, ZIP of a built frontend | **Not done.** |

## Setup
1. Create a Supabase project. Run `database/migrations/001_schema.sql` in the SQL editor.
2. `cd server && cp .env.example .env` and fill `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `ALLOWED_ORIGINS`, optionally `OPENAI_API_KEY`.
3. `npm install && npm start` (Node 18+). Check `GET /api/health`.
4. `npm test` runs the offline tests.
Never commit `.env`. The service-role key is not needed by the API; keep it out of the browser.

## Security design
- Every request except `/api/health` needs `Authorization: Bearer <Supabase JWT>`, verified with Supabase Auth.
- Queries use a per-request client carrying the user's JWT, so Postgres RLS enforces ownership (policies `*_own`; linked rows must reference the same user's email).
- zod validation on all inputs, strict schemas, 100 kB body limit, Helmet, CORS allow-list, global and AI rate limits, no headers/body/query in logs, generic 500 errors.
- Cross-user isolation is **designed via RLS but not verified against a live database.**

## API (all JSON, `{data}` or `{error:{message,code,details}}`)
`GET /api/health` · `GET|PATCH /api/me` · `GET /api/emails?q&category&priority&unread&status&sort&limit&offset` · `GET|PATCH|DELETE /api/emails/:id` · `POST /api/emails/:id/process` (OpenAI; cached in `ai_processing`) · `GET /api/approvals` · `POST /api/approvals/:id/decision {decision,reply?}` · `GET|POST /api/follow-ups` · `PATCH|DELETE /api/follow-ups/:id` · `GET /api/notifications` · `POST /api/notifications/read-all` · `POST /api/notifications/:id/read` · `GET /api/activity` · `GET|PUT /api/settings` · `GET /api/analytics/summary`.
Errors: 400 validation, 401 auth, 404, 409 already decided, 429 rate limit, 502/503/504 AI or auth service problems.

## Tests (run, offline)
`server/tests/api.test.js`: health flags, 401 on all protected routes, bad auth header, 503 when auth service unreachable, CORS allow/deny, security headers, 404, oversized body.

## Authentication (frontend)
Files: `frontend/auth.js` (Supabase client, validation, error mapping, token-bearing `AV.api()`), `frontend/config.js` (public config), `frontend/vendor/supabase.js` (supabase-js UMD, vendored).
1. In Supabase: Authentication -> Providers -> Email enabled. Decide whether "Confirm email" is on (if on, new users must confirm before signing in; the app tells them so). Add your frontend URL under Authentication -> URL Configuration.
2. Edit `frontend/config.js`: `SUPABASE_URL`, `SUPABASE_ANON_KEY` (public anon key only), `API_URL`.
3. Backend `server/.env`: same `SUPABASE_URL`/`SUPABASE_ANON_KEY`; `ALLOWED_ORIGINS` must contain the exact frontend origin.
4. Serve the frontend over http (not `file://`): `npx serve frontend -l 5173`, then start the API (`cd server && npm start`).
Behaviour: sign-up, sign-in, logout, session restored after refresh, protected `#app-*` routes redirect to `#start`, signed-in users skip `#start`, backend 401 signs the user out, 403/503/network errors show clear messages. If config is missing the forms say so; demo mode still works. Signed-in users currently see **generated sample data**; their mailbox is not connected.
Limitations: the password-reset email can be requested, but a screen to set the new password is not built. Google sign-in is not offered.

## Auth tests
- `cd server && npm test` (13 tests incl. backend token checks against a mock Supabase)
- `node --test frontend/tests/auth.test.js` (7 unit tests)
- `pip install playwright && playwright install chromium && python3 tests/e2e/e2e_auth.py` (11 browser tests; mock Supabase + real API + real frontend)
All three use a **mock** Supabase. Real Supabase has not been tested.
