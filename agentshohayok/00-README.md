# Agent-Shohayok — Feature শেখার গাইড

এই folder-এ project-এর ১০টা core feature আলাদা আলাদা file-এ বিস্তারিত ব্যাখ্যা করা হয়েছে (বাংলায়, tech term English-এ)। প্রতিটা file-এর শেষে ৫টা করে interview question ও উত্তর আছে।

**পড়ার ক্রম (সহজ → জটিল):**

| # | Feature | কী শিখবে |
|---|---------|----------|
| 01 | [Authentication ও Session](01-authentication-session.md) | login, PIN hashing (scrypt), cookie session, rate limiting |
| 02 | [Recharge](02-recharge.md) | DB transaction, race condition, idempotency, refund |
| 03 | [Balance Transfer](03-balance-transfer.md) | double-entry, atomic debit+credit, phone normalization |
| 04 | [Add Money](04-add-money.md) | approval workflow (maker-checker), status state machine |
| 05 | [Wallet ও Ledger](05-wallet-ledger.md) | double-entry accounting, Decimal money, immutable ledger |
| 06 | [KYC Verification](06-kyc-verification.md) | compliance workflow, human-in-the-loop, audit fields |
| 07 | [Admin ও RBAC](07-admin-rbac.md) | role-based access, defense in depth, privilege escalation |
| 08 | [Internationalization (i18n)](08-internationalization-i18n.md) | key-based translation, reactive switch, locale formatting |
| 09 | [State Management (Zustand)](09-state-management-zustand.md) | global state, selector, hydration, persist |
| 10 | [Routing ও Middleware](10-routing-middleware-security.md) | App Router, route group, security headers |

**Tech Stack:** Next.js 15 (App Router) · React 19 · TypeScript · Prisma · PostgreSQL · Zustand · TailwindCSS · Zod · Docker · Playwright

> পরামর্শ: প্রথমে file-টা পড়ো, তারপর project-এর আসল code (উল্লেখ করা file path) খুলে মিলিয়ে দেখো। শেষে interview question-গুলোর উত্তর নিজে মুখে বলার চেষ্টা করো — বলতে পারলে বুঝেছ।
