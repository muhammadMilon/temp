# Agent-Shohayok — Full Project Overview
### Retailer Fintech Platform | Mobile Recharge & Retailer Management System

---

## 1. Project Introduction

**Agent-Shohayok** is a Bangladeshi retailer fintech platform built for mobile recharge dealers and agents. Founded at Sonahat Landport Area, Kurigram, this dealership network digitalizes recharge operations, fund management, and customer tracking for retailers across Bangladesh.

| Item | Details |
|---|---|
| **Target Users** | Mobile recharge dealers & agents |
| **Language Support** | Bengali + English (dynamic switch) |
| **Theme** | Light & Dark mode |
| **Device** | Mobile-first, fully responsive |
| **Version** | v5.2.0 |
| **Owner** | Muhammad Lutfor — Sonahat Landport Area, Kurigram |

---

## 2. Tech Stack

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| **Next.js** | 15.3.3 | App Router, SSR, API Routes |
| **React** | 19.2.0 | UI Library |
| **TypeScript** | 5.8.3 | Type Safety |
| **Tailwind CSS** | 4.2.1 | Utility-first styling |
| **Framer Motion** | 12.40.0 | Animations & page transitions |

### UI Components
| Technology | Purpose |
|---|---|
| **shadcn/ui** | Component library (Button, Card, Dialog, Sheet, Tabs, etc.) |
| **Radix UI** | Headless UI primitives — accessibility-first |
| **Lucide React** | 0.575.0 — Icon library |
| **Recharts** | 2.15.4 — Charts & data visualization |
| **Sonner** | Toast notifications |
| **cmdk** | Command palette |
| **Vaul** | Drawer component |
| **input-otp** | OTP input component |

### State Management
| Technology | Purpose |
|---|---|
| **Zustand** | 5.0.14 — Global state (auth, theme, profile, recharge-pin) |
| **TanStack Query** | 5.83.0 — Server state, caching, mutations |
| **localStorage** | Offline queue, sidebar state persistence |

### Backend / API
| Technology | Purpose |
|---|---|
| **Next.js API Routes** | RESTful API endpoints (App Router) |
| **Prisma ORM** | 6.19.3 — Type-safe database access layer |
| **PostgreSQL (NeonDB)** | Production database (serverless) |
| **Upstash Redis** | Session caching, rate limiting |

### Forms & Validation
| Technology | Purpose |
|---|---|
| **React Hook Form** | 7.71.2 — Form state management |
| **Zod** | 3.24.2 — Schema validation |
| **@hookform/resolvers** | RHF + Zod integration |

### Internationalisation
| Technology | Purpose |
|---|---|
| **Custom i18n System** | `src/lib/i18n/` — Bengali & English translations |
| **Zustand LangStore** | Language preference persistence |

### Dev Tools
| Technology | Purpose |
|---|---|
| **ESLint** + **Prettier** | Code quality & formatting |
| **Prisma Studio** | Database GUI |
| **TypeScript** | Strict type checking |

---

## 3. Database Schema

**Database:** PostgreSQL via NeonDB (Serverless)
**ORM:** Prisma 6.19.3

### Core Models

```
User
 ├── id, phone (unique), pinHash, role, isActive, isFrozen
 ├── Profile      (fullName, shopName, address, district, nidNumber)
 ├── Wallet       (balance, openingBalance, frozenAmount, monthlyGoal)
 ├── Kyc          (nidNumber, status: PENDING|APPROVED|REJECTED)
 ├── Transaction[]     (recharge, transfer, add_money, refund)
 ├── Transfer[]        (sender → receiver by phone)
 ├── AddMoneyRequest[] (bkash, nagad, rocket, bank_transfer)
 ├── CustomerContact[] (agent-saved customer names & notes)
 ├── Session[]         (tokenHash, expiresAt)
 ├── Notification[]
 ├── AuditLog[]
 ├── Referral[]        (maker/user relationship)
 └── SystemLog[]
```

### Enums
| Enum | Values |
|---|---|
| `UserRole` | `AGENT`, `ADMIN`, `SUPER_ADMIN` |
| `TxStatus` | `PENDING`, `PROCESSING`, `SUCCESS`, `FAILED`, `CANCELLED`, `REFUNDED`, `HOLD` |
| `OperatorCode` | `GP`, `ROBI`, `AIRTEL`, `BANGLALINK`, `TELETALK`, `SKITTO`, `UNKNOWN` |
| `AddMoneyMethod` | `BKASH`, `NAGAD`, `ROCKET`, `BANK_TRANSFER` |
| `KycStatus` | `PENDING`, `APPROVED`, `REJECTED` |
| `LedgerType` | `DEBIT`, `CREDIT` |
| `AuditAction` | 20 actions (LOGIN, RECHARGE, TRANSFER, KYC, etc.) |
| `RechargeProviderName` | `MOCK`, `SUCCESS_TOPUP`, `ROBI_API`, `GP_API`, etc. |

### Transaction Flow
```
User Request
  → Transaction.create  (status: PENDING)
  → LedgerEntry.create  (DEBIT, balance before/after snapshot)
  → Wallet.update       (balance -= amount)
  → RechargeRequest.create (queued for provider)
  → Provider API call   (mock / real)
  → Transaction.update  (status: SUCCESS / FAILED)
  → AuditLog.create     (immutable record)
```

---

## 4. Project Structure

```
src/
├── app/
│   ├── (app)/                    # Protected Agent Routes
│   │   ├── dashboard/            # Main dashboard
│   │   ├── recharge/             # Mobile recharge
│   │   ├── add-money/            # Fund addition requests
│   │   ├── transfer/             # Balance transfer
│   │   ├── customers/            # Customer management
│   │   ├── history/              # Transaction history
│   │   ├── analytics/            # Business analytics
│   │   ├── notifications/        # Notifications
│   │   ├── policies/             # Legal policies
│   │   ├── about/                # About page
│   │   └── settings/             # Profile, KYC, preferences
│   ├── admin/                    # Admin Panel
│   │   └── page.tsx              # Dashboard + Reports + Add Money + Monitoring
│   ├── auth/                     # Authentication
│   │   ├── phone/                # Phone number entry
│   │   ├── otp/                  # OTP verification (signup only)
│   │   ├── pin-login/            # PIN login (existing users)
│   │   └── pin-create/           # PIN creation (new users)
│   ├── home/                     # Public landing page
│   └── api/                      # Next.js API Routes
│       ├── auth/                 # check, login, register
│       ├── recharge/             # POST recharge, GET history
│       ├── transfer/             # POST transfer
│       ├── add-money/            # POST add money request
│       ├── wallet/               # GET wallet balance
│       ├── dashboard/            # GET dashboard stats
│       ├── customers/            # GET list, PUT contact save
│       ├── user/profile/         # GET/PUT user profile
│       └── admin/
│           ├── overview/         # GET full system stats (real DB)
│           ├── reports/          # GET daily financial report (real DB)
│           └── add-money/        # GET pending, PATCH approve/reject
│
├── components/
│   ├── ui/                       # shadcn/ui components (30+)
│   ├── app-shell.tsx             # Agent layout (sidebar + header + bottom nav)
│   ├── admin-shell.tsx           # Admin layout (sidebar + header)
│   ├── auth-guard.tsx            # Protects agent routes
│   ├── admin-guard.tsx           # Protects admin routes
│   ├── logo.tsx                  # Brand logo component
│   ├── operator-badge.tsx        # Telecom operator badge
│   └── user-avatar.tsx           # User avatar with initials
│
├── features/
│   └── recharge/
│       └── recharge-content.tsx  # Full recharge form with offline support
│
├── lib/
│   ├── api/                      # Client-side API functions
│   │   ├── admin.api.ts          # Admin data fetching
│   │   ├── auth.api.ts           # Auth check/login
│   │   ├── customer.api.ts       # Customer CRUD
│   │   ├── recharge.api.ts       # Recharge submit & history
│   │   └── analytics.api.ts      # Analytics data
│   ├── store/                    # Zustand stores
│   │   ├── auth.ts               # Auth state + impersonation
│   │   ├── theme.ts              # Light/Dark/System theme
│   │   ├── profile.ts            # Cached profile data
│   │   └── recharge-pin.ts       # Session-scoped recharge PIN
│   ├── i18n/
│   │   ├── index.ts              # useT() hook, useLangStore()
│   │   └── translations.ts       # All BN/EN translation keys (100+)
│   ├── services/
│   │   ├── operators.ts          # Phone → operator auto-detection
│   │   ├── ledger.ts             # Wallet ledger service
│   │   ├── audit-log.ts          # Audit logging service
│   │   └── storage.ts            # localStorage utility + delay()
│   ├── providers/recharge/       # Pluggable recharge API providers
│   ├── db.ts                     # Prisma client singleton
│   ├── redis.ts                  # Upstash Redis client
│   ├── api-helpers.ts            # getRequestUser(), unauthorized()
│   └── types/                    # Shared TypeScript types
│
└── prisma/
    └── schema.prisma             # Full production DB schema (13 models)
```

---

## 5. Features

### 5.1 Authentication

```
Login Flow:
  Enter Phone → Check DB → Registered?
                               ├── Yes → Enter PIN → Match? → Dashboard
                               │                   └── No  → "Wrong PIN"
                               └── No  → "No account found, please sign up"

Signup Flow:
  Enter Phone → Not Registered → OTP Verification → Create 5-digit PIN → Dashboard
```

- Custom 11-digit phone input (box-by-box entry)
- Auto-detect mobile operator on partial input
- Demo OTP: `123456`
- 5-digit agent PIN (stored in DB)
- 6-digit admin PIN (hardcoded: `123456`)
- Auth state persisted with Zustand (`localStorage key: rp.auth`)

---

### 5.2 Agent Dashboard

- Real-time wallet balance (fetched from DB)
- Today's recharge volume, monthly revenue
- Recent transactions list
- Quick action buttons (Recharge, Add Money, Transfer)
- Operator breakdown chart (Recharts)
- Monthly goal progress bar

---

### 5.3 Mobile Recharge

- 11-digit phone number input
- **Auto operator detection** — GP, Robi, Airtel, Banglalink, Teletalk, Skitto
- Customer auto-suggest (search by name or phone)
- **Smart Suggestion** — shows customer's last recharge amount & frequency
- **Package expiry warning** — predicts if customer's package is about to expire
- **Quick amount picker** — 3 offer cards (GP 29, Robi 49, BL 99) that auto-fill amount
- Keyboard flow: Phone `Enter` → Amount field → `Enter` → Recharge
- **Recharge Security PIN** — verified once per session (separate from login PIN)
- **Offline support** — queues transactions when offline, auto-syncs on reconnect
- Minimum balance check (BDT 500)
- Favorites list and recent numbers quick-dial

---

### 5.4 Add Money

- Supports bKash, Nagad, Rocket, Bank Transfer
- Submit transaction ID as proof
- Admin reviews and approves/rejects
- Balance credited to wallet after approval

---

### 5.5 Balance Transfer

- Transfer from agent to another agent or user
- Receiver phone validation
- Optional note/reason field
- PIN verification before transfer
- Transfer history (collapsible accordion)
- Success confirmation screen with receipt

---

### 5.6 Customer Management

- **DB-backed** — customer list built from recharge transaction history
- `CustomerContact` model — per-agent saved customer name & note
- Customer segments: Regular, VIP, At Risk, New
- Favorite toggle (starred customers)
- Search by name or phone number
- Shows: last recharge date, recharge count, average amount
- Edit dialog — save custom name & notes (persisted to DB)

---

### 5.7 Transaction History

- Full transaction list with status
- Filter by status (success, failed, pending)
- Date-wise grouping
- Print / Export support

---

### 5.8 Analytics

- Daily / Weekly / Monthly revenue charts
- Operator-wise recharge breakdown
- Customer count trends
- Interactive graphs powered by Recharts

---

### 5.9 Notifications

- System notifications (DB-backed)
- Transaction success/failure alerts
- Add money approval notifications

---

### 5.10 Settings

| Sub-page | Details |
|---|---|
| Profile | Name, shop, address, district update |
| KYC | NID number, photo upload, status tracking |
| Policies | Terms, Privacy, Refund, KYC, AML policies |
| About Owner | Founder information |
| Language | Bengali / English toggle |
| Theme | Light / Dark / System |

---

### 5.11 Admin Panel

**Admin phone:** `01864697343` | **PIN:** `123456`

#### Overview Tab (Real DB Data)
- Total users, agents, active agents
- KYC pending / approved / rejected
- Total recharge volume (monthly)
- Transfer volume (today)
- System revenue (today / weekly / monthly)
- Fraud alerts (transfers ≥ BDT 50,000)
- Total wallet balance across all agents
- Top referrers leaderboard

#### Reports Tab (Real DB Data)
- Today's recharge volume & profit
- Operator breakdown pie chart
- 24-hour hourly activity bar chart
- 7-day daily trends line chart

#### Add Money Tab
- Pending add money request list
- Approve / Reject / Hold actions with admin notes
- Auto-refreshes every 30 seconds

#### Monitoring Tab
- System health indicators
- Service status panels

#### Agent View Mode
- Admin can impersonate any agent and view their dashboard
- Yellow "Viewing as Agent" banner with "Exit Agent View" button

---

## 6. API Endpoints

### Auth APIs
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/check` | Check if phone is registered |
| `POST` | `/api/auth/login` | Verify PIN & create session |
| `POST` | `/api/auth/register` | Register new user |

### Agent APIs
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/wallet` | Get wallet balance |
| `GET` | `/api/dashboard` | Dashboard stats |
| `POST` | `/api/recharge` | Submit recharge request |
| `GET` | `/api/recharge/history` | Recharge transaction history |
| `POST` | `/api/transfer` | Submit balance transfer |
| `POST` | `/api/add-money` | Submit add money request |
| `GET` | `/api/customers` | Customer list from transaction history |
| `PUT` | `/api/customers/[phone]` | Save customer name/note/favorite |
| `GET/PUT` | `/api/user/profile` | View / update user profile |

### Admin APIs
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/admin/overview` | Full system stats (real DB) |
| `GET` | `/api/admin/reports` | Daily financial report (real DB) |
| `GET` | `/api/admin/add-money` | Pending add money requests |
| `PATCH` | `/api/admin/add-money/[id]` | Approve / Reject / Hold a request |

---

## 7. Authentication Architecture

```
Every API request includes:
  Header: x-user-phone: 01XXXXXXXXX
                    ↓
           getRequestUser()  (src/lib/api-helpers.ts)
                    ↓
        DB Lookup → User upsert (create if first visit)
                    ↓
          Role assigned:
            01864697343 → ADMIN
            all others  → AGENT
```

- Demo mode: `x-user-phone` header acts as identity (no JWT needed)
- Production-ready: schema has Session model with token hash + expiry
- All protected routes return `401 Unauthorized` without valid phone header

### Zustand Auth Store (client-side)
```typescript
{
  phone: string,
  pin: string,
  role: "agent" | "admin" | null,
  isAuthenticated: boolean,
  impersonatingAsAgent: boolean,  // Admin viewing as agent mode
}
```

---

## 8. State Management

| Store | localStorage Key | Purpose |
|---|---|---|
| `useAuthStore` | `rp.auth` | Login state, logout, agent impersonation |
| `useThemeStore` | `rp.theme` | Light / Dark / System theme |
| `useLangStore` | `rp.lang` | Bengali / English language preference |
| `useProfileStore` | in-memory | Cached profile data |
| `useRechargePinStore` | in-memory | Session-scoped recharge PIN verification |

**TanStack Query** caches all server data:
| Query Key | Data |
|---|---|
| `["wallet"]` | Wallet balance |
| `["dashboard"]` | Dashboard stats |
| `["transactions"]` | Transaction history |
| `["customers"]` | Customer list |
| `["admin-overview"]` | Admin system stats |
| `["admin-daily-report"]` | Financial report |

---

## 9. Operator Detection System

`src/lib/services/operators.ts` — Auto-detects operator from phone prefix:

| Operator | Brand Color | Detected Prefixes |
|---|---|---|
| Grameenphone (GP) | `#E2001A` | 011, 013, 014, 017 |
| Robi | `#E2001A` | 018 |
| Banglalink | `#F7941D` | 019 |
| Airtel | `#ED1C24` | 016 |
| Teletalk | `#2E8B57` | 015 |
| Skitto | `#FF4F8B` | 013 (subset) |

- Detection starts from 3 digits typed
- Operator badge auto-appears in recharge input
- Color-coded badge in customer lists

---

## 10. UI/UX Features

### Responsive Layout
- **Mobile:** Bottom navigation bar (5 items), hamburger menu → side drawer
- **Desktop:** Fixed collapsible sidebar, sticky header with page title
- Sidebar visibility uses plain CSS `@media` (bypasses Tailwind v4 cascade bug)
- Sidebar collapse state persisted in localStorage

### Theme System
- Light / Dark / System — uses `oklch()` color space (perceptual uniformity)
- Custom CSS variables for all semantic colors
- Sidebar has distinct dark background (`oklch(0.2 0.02 255)`)
- Smooth transitions on theme switch

### Animations
- Framer Motion page transitions (`opacity + y` spring)
- `AnimatePresence` for conditional renders
- Smooth sidebar collapse / expand

### Accessibility
- All Radix UI components (keyboard navigation, ARIA labels)
- `DialogTitle` in every Dialog/Sheet (screen reader compliance)
- `sr-only` labels for visual-only UI elements

### Offline Support
- Recharge requests queued in localStorage when offline
- Online event listener triggers automatic background sync
- Toast notification shows offline/online status changes

---

## 11. Security

| Feature | Details |
|---|---|
| **PIN Storage** | Stored in DB (production: bcrypt hash) |
| **Recharge PIN** | Session-scoped — verified once per session, separate from login PIN |
| **Admin Guard** | Route-level `ADMIN` role check on every request |
| **Auth Guard** | `isAuthenticated` check before all protected pages |
| **Audit Log** | Every critical action recorded in DB (immutable) |
| **Fraud Detection** | Transfers ≥ BDT 50,000 auto-flagged as suspicious |
| **Wallet Ledger** | Every balance change creates a `LedgerEntry` (balance before/after snapshot) |
| **Idempotency** | Unique `trxId` prevents duplicate recharge submissions |
| **Input Validation** | Zod schema validates every API input |
| **Account Freeze** | Admin can freeze wallets (`isFrozen: true`) |

---

## 12. Internationalisation (i18n)

`src/lib/i18n/translations.ts` — All UI text stored as key-value pairs:

```typescript
// Sample translation keys
nav_dashboard:   { bn: "ড্যাশবোর্ড",        en: "Dashboard"         }
recharge_title:  { bn: "রিচার্জ করুন",       en: "Recharge"          }
auth_pin_wrong:  { bn: "ভুল PIN!",            en: "Wrong PIN!"        }
nav_add_money:   { bn: "অ্যাড মানি",          en: "Add Money"         }
nav_transfer:    { bn: "ট্রান্সফার",           en: "Transfer"          }
```

- `useT()` hook — access any translation in any component
- `useLangStore` — global language switch with Zustand persist
- `formatCurrency(amount, lang)` — renders `৳ ১,০০০` (BN) or `৳ 1,000` (EN)
- `formatNum(number, lang)` — Bengali numeral support (০১২৩...)
- Language switch takes effect instantly, no page reload needed

---

## 13. Database Flow Diagrams

### Recharge Flow
```
POST /api/recharge
  → getRequestUser()           Identify agent by phone header
  → Generate unique trxId      Idempotency key
  → Transaction.create         status: PENDING
  → LedgerEntry.create         DEBIT, capture balance before/after
  → Wallet.update              balance -= amount
  → RechargeRequest.create     Queue for provider
  → Call provider API          Mock (demo) / Real (production)
  → Transaction.update         status: SUCCESS or FAILED
  → AuditLog.create            Immutable record
  → Response to client
```

### Add Money Approval Flow
```
PATCH /api/admin/add-money/[id]  { action: "approve" }
  → Verify requester is ADMIN
  → AddMoneyRequest.update     status: SUCCESS
  → LedgerEntry.create         CREDIT
  → Wallet.update              balance += amount
  → Notification.create        Alert sent to agent
  → AuditLog.create
  → Response to admin
```

---

## 14. Recharge Provider Architecture

`src/lib/providers/recharge/`

Pluggable provider pattern — swap real API without changing business logic:

```typescript
interface RechargeProvider {
  name: RechargeProviderName;
  recharge(
    phone:    string,
    amount:   number,
    operator: OperatorCode
  ): Promise<{ success: boolean; providerTrxId?: string; error?: string }>;
}
```

| Provider | Status | Notes |
|---|---|---|
| `MockProvider` | Active (demo) | Always returns success after delay |
| `SuccessTopUp` | Planned | Real API integration |
| `GP API` | Planned | Grameenphone direct API |
| `Robi API` | Planned | Robi direct API |
| `SSL Wireless` | Planned | Aggregator |

---

## 15. Page Routing

```
/                    Landing page (redirect logic)
/home                Public home (Login / Sign Up buttons)
/auth/phone          Phone input  (?intent=login|signup)
/auth/otp            OTP verify   (signup flow only)
/auth/pin-login      PIN entry    (returning users)
/auth/pin-create     Create PIN   (new users after OTP)

/dashboard           Agent dashboard          [Auth required]
/recharge            Mobile recharge          [Auth required]
/add-money           Add money request        [Auth required]
/transfer            Balance transfer         [Auth required]
/customers           Customer management      [Auth required]
/history             Transaction history      [Auth required]
/analytics           Business analytics       [Auth required]
/notifications       Notifications            [Auth required]
/policies            Policies & legal         [Auth required]
/about               About us                 [Auth required]
/settings            Settings hub             [Auth required]
/settings/profile    Edit profile             [Auth required]
/settings/kyc        KYC submission           [Auth required]

/admin               Admin dashboard          [Admin only]
```

---

## 16. What Makes It Unique

1. **Bangladesh-specific** — BD phone number validation, Bengali numerals, local operators with correct prefix mapping
2. **Offline Recharge Support** — Queue transactions locally, auto-sync when internet returns
3. **Smart Customer Intelligence** — Package expiry prediction based on average recharge interval from history
4. **Session-scoped Recharge PIN** — Extra security layer on top of login PIN, verified once per session
5. **Admin Agent Impersonation** — Admin can view and test the platform as any agent
6. **Ledger-based Accounting** — Every balance change has a full audit trail (balance before/after snapshot)
7. **Dual Language** — Full Bengali + English throughout, toggle instantly without reload
8. **Pluggable Provider System** — Swap recharge API provider without changing business logic
9. **Real-time Admin Stats** — DB-backed live data, 30-second auto-refresh on add money queue
10. **Idempotent Transactions** — Unique transaction ID prevents duplicate charges

---

## 17. Project Statistics

| Metric | Count |
|---|---|
| Total source files | ~120+ |
| API Endpoints | 17 |
| Database Models | 13 |
| Database Enums | 8 |
| shadcn/ui Components | 30+ |
| Translation Keys | 100+ |
| Zustand Stores | 5 |
| App Pages/Routes | 20+ |
| Recharge Operators | 6 |
| Lines of Code (approx.) | ~8,000+ |

---

## 18. Roadmap

| Feature | Status |
|---|---|
| Real SMS OTP (SMS Gateway) | Planned |
| Real Recharge API Integration | Planned |
| KYC Document Image Upload | Planned |
| Push Notifications | Planned |
| Agent Referral System | DB schema ready |
| Commission Slab Management | Planned |
| Multi-district Agent Network | Planned |
| Mobile App (React Native) | Future |
| Redis Job Queue (Worker) | Infrastructure ready |
| bcrypt PIN hashing | DB schema ready |

---

## 19. Developer Information

| | |
|---|---|
| **Developer** | Muhammad Milon |
| **Platform Owner** | Muhammad Lutfor |
| **Location** | Sonahat Landport Area, Kurigram, Bangladesh |
| **Helpline** | 01864697343 |
| **Email** | support@agent-shohayok.com |
| **Tech Stack** | Next.js 15 + Prisma + PostgreSQL + Tailwind CSS v4 |
| **Version** | v5.2.0 (June 2026) |

---

*Agent-Shohayok — Built for Bangladesh's retailers, by Bangladesh's retailers.*
