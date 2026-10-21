# CampusBite - Security Architecture & STRIDE Threat Model
**Standard Compliance:** OWASP Top 10 (2021), OWASP ASVS v4.0.3 Level 2  
**Target System:** CampusBite Mobile-First College Canteen PWA

---

## 1. STRIDE Threat Model & Mitigations

| STRIDE Category | Threat Scenario | Architectural Mitigation | Code Location |
| :--- | :--- | :--- | :--- |
| **S - Spoofing** | Attacker impersonates an enrolled student or college canteen staff | Strict JWT in `httpOnly`, `Secure`, `SameSite=Strict` cookies. Refresh token rotation with cryptographic hash in database. Passwords hashed with `Argon2id` (64MB memory cost, parallelism=4). Multi-factor authentication (TOTP) supported. Roster-bound registration requiring 6-digit OTP verification. | `src/lib/auth/jwt.ts`<br>`src/lib/auth/password.ts`<br>`src/app/api/auth/login/route.ts` |
| **T - Tampering** | Malicious student alters item prices, totals, or stock counts in cart payload | Zero-trust server-side recalculation. Client prices, subtotal, and tax are discarded upon checkout. Item prices and add-on adjustments are resolved strictly from the database. Atomic stock decrement via database transaction prevents race conditions and overselling. | `src/app/api/orders/route.ts`<br>`src/lib/db/repository.ts` |
| **R - Repudiation** | Staff member denies changing food prices or redeeming an order ticket | Comprehensive, tamper-evident audit logging (`AuditLog` and `PriceHistory` tables). Captures actor user ID, role, IP address, user-agent, timestamp, and payload diff for every administrative action. Ticket redemption records staff ID and timestamp. | `src/lib/security/audit.ts`<br>`src/app/api/tickets/redeem/route.ts`<br>`src/app/api/admin/pricing/route.ts` |
| **I - Information Disclosure** | Student enumerates college IDs or snoops on another student's order/ticket (IDOR) | Generic authentication failure messages ("Invalid College ID or password") with timing-safe checks. Non-sequential UUIDs for orders and cryptographic ticket codes (`CB-YYMMDD-XXXX`). Strict object-level ownership checks (`studentId === session.user.id`) enforced on every endpoint. DB Row Level Security policies. | `src/lib/auth/rbac.ts`<br>`src/app/api/orders/[id]/route.ts`<br>`prisma/rls_policies.sql` |
| **D - Denial of Service** | Bot script floods login endpoint or creates hundreds of uncollected orders | In-memory token bucket rate limiting (10 attempts/min on auth, 60/min general). Account lockout (5 failed attempts = 15-minute lock). Limit of maximum 3 active uncollected orders per student. Strict Zod quantity limits (max 20 per item). | `src/lib/security/rate-limiter.ts`<br>`src/app/api/orders/route.ts` |
| **E - Elevation of Privilege** | Student tampers with JWT role claim or calls admin endpoints | Server-side role enforcement via `requireAuth(req, ["ADMIN", "STAFF"])` on every route handler. JWT signatures verified with HS256 secret. Instant session revocation when student ID is banned. | `src/lib/auth/rbac.ts`<br>`src/app/api/admin/users/status/route.ts` |

---

## 2. OWASP Top 10 Compliance Verification

### A01:2021 – Broken Access Control
- [x] **Server-side RBAC:** Every API route verifies role membership using `requireAuth`.
- [x] **IDOR Protection:** `checkObjectOwnership` validates student ID on order details, ticket display, and review submissions.
- [x] **Database Row Level Security (RLS):** SQL policies provided in `prisma/rls_policies.sql`.
- [x] **Immediate Revocation:** Deactivating/banning a student instantly revokes all active refresh tokens in the database.

### A02:2021 – Cryptographic Failures
- [x] **Argon2id Password Hashing:** 64 MB memory cost, 3 time iterations, 4 parallel threads, fallback to bcrypt (cost factor 12).
- [x] **CSPRNG OTP Tokens:** 6-digit cryptographic random integer generator (`crypto.randomInt`).
- [x] **OTP Salted Digests:** Plaintext OTPs are never stored; only salted SHA-256 HMAC hashes exist in the database.
- [x] **Strict Cookie Security:** `httpOnly: true`, `secure: true` (in production), `sameSite: "strict"`, `path: "/"`.

### A03:2021 – Injection
- [x] **Parameterized Queries:** Prisma ORM prepared statements eliminate SQL injection vulnerabilities.
- [x] **Zod Input Schema Validation:** Strict validation schemas on both client and server reject SQL/NoSQL metacharacters.
- [x] **Zero `eval`:** No dynamic code evaluation or unsanitized DOM rendering.

### A04:2021 – Insecure Design
- [x] **Atomic Concurrency:** Stock deduction executed inside serializable transactions (`SELECT FOR UPDATE` / atomic decrement).
- [x] **Idempotency Keys:** Network retries with duplicate idempotency keys return existing orders without double billing.
- [x] **Single-Use Ticket Lock:** Pickup tickets cannot be redeemed more than once (`isRedeemed` atomic transition).

### A05:2021 – Security Misconfiguration
- [x] **Security Headers:** Strict CSP, HSTS (`max-age=63072000; includeSubDomains; preload`), `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Permissions-Policy`.
- [x] **Powered-By Header Disabled:** `poweredByHeader: false` in `next.config.mjs`.
- [x] **Non-Root Docker User:** Docker container runs as unprivileged user `nextjs` (UID 1001).

### A06:2021 – Vulnerable and Outdated Components
- [x] **Fixed Dependencies:** Stable Prisma 5.22.0, patched Next.js latest release.
- [x] **Committed Lockfile:** `package-lock.json` committed.

### A07:2021 – Identification and Authentication Failures
- [x] **College Roster Verification:** Only students listed on the administrator's uploaded CSV roster can register.
- [x] **Password Complexity:** Minimum 10 characters, checked against common passwords blacklist, requires uppercase, lowercase, digits, and special symbols.
- [x] **Rate Limiting & Account Lockout:** 5 failed logins lock account for 15 minutes.
- [x] **Token Rotation:** Refresh tokens rotate on every access renewal.

### A08:2021 – Software and Data Integrity Failures
- [x] **Payment Webhook Verification:** Razorpay/UPI webhooks verify HMAC SHA-256 signatures via timing-safe byte comparison (`crypto.timingSafeEqual`).

### A09:2021 – Security Logging and Monitoring Failures
- [x] **Audit Trail:** Privileged actions, failed logins, roster imports, price adjustments, and bans recorded in `AuditLog`.

### A10:2021 – Server-Side Request Forgery (SSRF)
- [x] **Controlled URLs:** Image inputs validated through strict URL schemas.

---

## 3. Automated Test Verification
Run test suite:
```bash
npx tsx --test tests/auth.test.js
npx tsx --test tests/order-checkout.test.js
npx tsx --test tests/security-rbac.test.js
```
- Auth & Password Security: **PASS**
- Concurrency & Stock Race Conditions: **PASS**
- RBAC Boundaries & IDOR Isolation: **PASS**
- Ticket Replay & Double Redemption: **PASS**
