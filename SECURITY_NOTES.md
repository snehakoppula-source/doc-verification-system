# Security Review & Fixes — Doc Verification System

This document lists every issue found in the original codebase and exactly what was
changed to fix it. Use it for your project report / viva — it explains *why*, not just *what*.

## Critical

1. **NoSQL injection in login/register** (`userController.js`)
   `User.findOne({ email })` used `req.body.email` directly. Since the body is parsed
   as JSON, an attacker could send `{"email": {"$ne": null}, "password": {"$ne": null}}`
   and match an arbitrary user, bypassing authentication.
   **Fix:** added `utils/validators.js` and require `email`/`password`/etc. to be
   strings of sane length/format *before* they touch a query. Also added a global
   `middleware/sanitize.js` that strips any `$`-prefixed or dotted key from
   `req.body`/`req.params` as defense in depth.

2. **Wide-open CORS** (`index.js`)
   `app.use(cors())` reflected any origin, letting any website make requests to
   this API on a logged-in user's behalf.
   **Fix:** CORS is now restricted to an explicit allow-list read from `CORS_ORIGIN`.

3. **Unhandled crypto exceptions crash requests with a 500 + stack trace**
   (`companyController.js`, `instituteController.js`) — a malformed signature or
   corrupted public key made `crypto.createVerify(...).verify(...)` throw, which
   was never caught, so Express's default handler returned a raw stack trace.
   **Fix:** wrapped verification in `try/catch`, return a clean `400`.

4. **Institution-code hijacking** (`userController.js`, `instituteCertController.js`)
   Nothing stopped a second user from registering as `Institution` with a code
   that already belonged to a real institution, then uploading a new public key
   and effectively impersonating it (all future "signed" documents would verify
   against the attacker's key).
   **Fix:** registration now rejects an `institutionCode` already used by an
   existing `Institution` account. *(Note: for production use you'd want an
   admin-approval step for new institutions rather than self-declared codes —
   flagging this as a remaining architectural limitation.)*

## High

5. **No rate limiting on auth endpoints** → unrestricted brute-force/credential
   stuffing against `/api/login`. **Fix:** `express-rate-limit` on `/api/login`
   and `/api/register` (20 requests / 15 min), plus a looser general limiter.

6. **No file-size/type limits on uploads** (`routes/institute.js`, `routes/company.js`)
   `multer({ storage: multer.memoryStorage() })` had no `limits`, so any
   authenticated user could upload huge files straight into server RAM (DoS),
   and any file type — not just PDFs.
   **Fix:** added `limits.fileSize` (10 MB for documents, 1 MB for PEM certs)
   and a `fileFilter` requiring `application/pdf` for document uploads.

7. **Unvalidated "public key" upload** (`instituteCertController.js`) — any text
   was accepted and stored as a PEM public key, only failing later (as an
   unhandled exception) the first time it was used to verify a signature.
   **Fix:** validate the upload matches a PEM `PUBLIC KEY` block before saving,
   and cap its size.

8. **Duplicate-key crash on PKC re-upload** (`instituteCertController.js`) — the
   model has a unique index on `institutionCode`, but the controller always did
   an `insert`, so uploading a second certificate for the same institute threw
   an unhandled Mongo duplicate-key error.
   **Fix:** switched to `findOneAndUpdate(..., { upsert: true })`.

9. **Weak/no password policy** (`userController.js`, `Register.jsx`) — any
   non-empty password was accepted. **Fix:** minimum 8 characters, enforced on
   both server (authoritative) and client (UX).

10. **JWT algorithm not pinned** (`middleware/auth.js`) — `jwt.verify(token, secret)`
    without an explicit `algorithms` option is best practice to avoid, since it
    leaves the door open to algorithm-confusion issues if the library or key
    material ever changes. **Fix:** both signing and verification now pin
    `HS256` explicitly.

## Medium / hardening

11. **No `helmet`** → missing standard security headers. **Fix:** added `helmet()`.
12. **No centralized error handler** → any thrown error (validation, cast,
    duplicate key, Multer, JWT) fell through to Express's default handler and
    leaked stack traces. **Fix:** `middleware/errorHandler.js` maps known error
    types to clean, minimal JSON responses; unexpected errors are logged
    server-side and returned as a generic `500`.
13. **No request body size limit** → `express.json({ limit: '1mb' })` added.
14. **No startup validation of required env vars** (`MONGODB_URI`, `JWT_SECRET`)
    → the app now exits immediately with a clear message if they're missing,
    instead of crashing confusingly later.
15. **Sensitive data in logs** — `console.log(signature)` in
    `instituteController.js` printed a full signature to server logs on every
    failed upload. Removed.
16. **Duplicate certificate records** — nothing stopped the same roll number
    being uploaded twice for an institute. Added a uniqueness check.
17. **Frontend inconsistency/bug risk** — every component called raw `axios`
    with its own hardcoded base URL and manually attached the auth header
    (easy to forget). **Fix:** centralized in `src/utils/axios.js` with a
    request interceptor that always attaches the token; components now use
    relative paths (`/login`, `/institute/upload`, etc.).

## Noted but not changed (needs a design decision, not just a patch)

- **JWT stored in `localStorage`** (`AuthContext.jsx`) is convenient but
  readable by any JavaScript running on the page, so an XSS bug anywhere in
  the app would allow token theft. The more robust pattern is an `httpOnly`
  cookie + CSRF token, but that requires backend session/cookie changes
  (SameSite config, CSRF middleware, logout-by-cookie-clear) which is a larger
  refactor than a security patch — worth doing before any real deployment,
  but left as-is here so the app's behavior doesn't change unexpectedly.
- **Self-declared institution codes** — see item 4 above. The mitigation added
  prevents the most obvious hijack, but a proper fix is admin-verified
  institution onboarding.

## What you need to do before running

1. `cd backend && cp .env.example .env` and fill in `MONGODB_URI` and a real
   `JWT_SECRET` (a random 64-byte hex string is fine).
2. `cd frontend && cp .env.example .env` (defaults already point at
   `http://localhost:5000`).
3. `cd backend && npm install` (installs the new `helmet` and
   `express-rate-limit` dependencies too).
4. `cd frontend && npm install`
5. Run MongoDB locally (or point `MONGODB_URI` at Atlas).
6. `cd backend && npm run dev` then, in another terminal, `cd frontend && npm run dev`.
