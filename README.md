# 🍔 CampusBite - College Canteen Ordering PWA

> Production-ready, mobile-first Progressive Web App (PWA) for college canteen ordering, pickup ticketing, and live order tracking. Inspired by the Domino's, McDonald's, and Burger King mobile ordering flows, engineered with zero-trust security and atomic inventory management.

---

## 🚀 Key Features

### 🎓 1. Student Experience (Mobile-First PWA)
- **Official College ID Login:** Only verified college students on the administrator's roster can create an account or log in.
- **First-Time Login Verification:** First login dispatches a 6-digit cryptographic OTP to the registered college email and forces permanent password creation (10+ characters, Argon2id hashing).
- **Interactive Menu Catalog:** Categories (*Snacks, Meals, Beverages, Combos, Desserts*), real-time search, Veg/Non-Veg filter, bestseller & spicy tags.
- **Food Customization Drawer:** Size variations, add-ons (extra cheese, patty, dips), and special cooking instructions.
- **Out-of-Stock Protection:** Items with 0 inventory are automatically disabled and badged as "Unavailable".
- **Server-Recalculated Checkout:** Server recalculates all prices and verifies stock inside a database transaction — client prices and totals are never trusted.
- **Unique Digital Pickup Pass:** Generates a non-guessable ticket code (e.g. `CB-261015-X7K2`) and QR code for pickup at the counter.
- **Live Order Progress Tracker:** Domino's/BK-style timeline (*Placed → Accepted → Preparing → Ready for Pickup → Collected*) with live updates and audio chime alerts when food is ready.
- **Verified Reviews & Feedback:** Post reviews only after collecting meals; direct complaint ticketing to canteen management.
- **Profile & Security Control:** View details, reset password via OTP, and revoke all active device sessions with one click.

### 👨‍🍳 2. Canteen Staff Operations
- **Live Kitchen Display System (KDS):** Real-time order queue with audio chime alerts on incoming orders.
- **Workflow State Management:** Single-click transitions (*Accept → Cook → Ready → Collected*) or decline with reason.
- **Counter QR Scanner & Redemption:** Scan or enter digital ticket codes. Implements atomic single-use redemption locks to eliminate double-claim fraud.

### 🛡️ 3. Canteen Administration
- **Menu Management:** Add, edit, archive items, categories, descriptions, image URLs, and customizations.
- **Pricing Manager:** Inline price editing, bulk adjustments, and tamper-evident price change history logs.
- **Atomic Stock Controller:** Daily stock reset button (morning replenishment to 50 units), manual inventory adjustments, and low-stock alerts.
- **Student Roster Management:** Upload student CSV roster (`collegeId,name,department,year,email,phone`), search students, and instant account deactivation kill-switch (revokes all active sessions).
- **Review & Feedback Moderation:** Approve, hide, and reply to student reviews and complaints.
- **Analytics & CSV Export:** Live revenue, order volume, average tray value, peak hours breakdown, top-selling items, and CSV export.
- **Audit Log Trail:** Tamper-evident log of every administrative and security event with actor, IP, timestamp, and payload diff.

---

## 🔑 Test Credentials (Seed Data)

| Role | College ID | Password | Notes |
| :--- | :--- | :--- | :--- |
| **Admin** | `ADMIN-001` | `AdminSecurePassword2026!` | Prof. Vikram Malhotra (Full administrative control) |
| **Staff** | `STAFF-001` | `StaffSecurePassword2026!` | Chef Ramesh Kumar (Kitchen Display Queue & Ticket Scanner) |
| **Verified Student** | `STU-2024-001` | `StudentSecurePass2026!` | Aarav Sharma (CS 3rd Year) |
| **First-Login Student** | `STU-2024-002` | *Triggers OTP* | Diya Patel (Tests first-time OTP verification flow) |

> 💡 **Testing First Login:** Log in with `STU-2024-002`. The app detects a first-time login from the official roster, dispatches a 6-digit OTP (printed directly to the server terminal console), and redirects to the verification screen to establish a permanent password.

---

## 🛠️ Tech Stack & Security Architecture

- **Frontend:** Next.js (App Router) + TypeScript + Tailwind CSS, installable PWA with Service Worker (`sw.js`) and manifest.
- **Backend:** Next.js Route Handlers with universal repository pattern.
- **Database:** PostgreSQL (Supabase / Neon / Docker) via Prisma ORM + local JSON repository fallback for instant zero-dependency testing.
- **Security & Cryptography:**
  - Password Hashing: `Argon2id` (64 MB memory cost, 4 parallelism threads)
  - Sessions: 15-minute access JWT + 7-day rotating refresh tokens in `httpOnly`, `Secure`, `SameSite=Strict` cookies
  - Defense-in-Depth Authorization: Server-side RBAC and strict object-level checks (IDOR prevention)
  - Concurrency: Atomic database transactions with row-level locks for stock deduction
  - Webhooks: Razorpay/UPI payment webhooks with HMAC SHA-256 signature checks
  - Security Headers: Content-Security-Policy, HSTS, X-Frame-Options: DENY, X-Content-Type-Options: nosniff

---

## 🏃 Quick Start (Local Development)

### 1. Prerequisites
- Node.js >= 18.0.0
- npm >= 9.0.0

### 2. Installation
```bash
# Clone or navigate to the directory
cd C:\Users\somya\.gemini\antigravity\scratch\campusbite

# Install dependencies
npm install
```

### 3. Environment Variables
Copy `.env.example` to `.env`:
```bash
cp .env.example .env
```

### 4. Run Development Server
```bash
npm run dev
```
Open **[http://localhost:3000](http://localhost:3000)** in your browser or mobile viewport emulator.

---

## 🧪 Running Automated Tests

CampusBite includes a test suite covering authentication, RBAC boundaries, input validation fuzzing, atomic stock concurrency, and IDOR protection:

```bash
# Run Auth & Cryptography Tests (Password rules, Argon2id, OTP, Rate limiter)
npx tsx --test tests/auth.test.js

# Run Order, Stock Concurrency & Ticket Redemption Tests
npx tsx --test tests/order-checkout.test.js

# Run Server-Side RBAC, Zod Input Validation & IDOR Tests
npx tsx --test tests/security-rbac.test.js
```

---

## 🐳 Production Deployment (Docker)

To run the complete production stack (Next.js Application + PostgreSQL) using Docker Compose:

```bash
docker compose up --build
```
The application will be accessible at `http://localhost:3000`.

### Database Migrations for Supabase / Neon:
1. Set `DATABASE_URL` in `.env` to your PostgreSQL connection string.
2. Run Prisma schema synchronization:
   ```bash
   npx prisma db push
   ```
3. Execute the Row Level Security policies:
   ```bash
   psql $DATABASE_URL -f prisma/rls_policies.sql
   ```
4. Run the seed script:
   ```bash
   node prisma/seed.js
   ```

---

## 📋 Security Documentation
Review the comprehensive **[SECURITY_CHECKLIST.md](./SECURITY_CHECKLIST.md)** for detailed ASVS verification and the complete STRIDE threat model.
