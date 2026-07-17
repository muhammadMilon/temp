# SkillVoyager.AI — Feature Teaching Docs

এই ফোল্ডারটা SkillVoyager.AI প্রজেক্টের **আসল source code** পড়ে বানানো ১০টি standalone শেখার ডকুমেন্ট — প্রতিটা ফিচার একদম বেসিক থেকে শুরু করে, কেন এভাবে বানানো হয়েছে (WHY) সেটা বোঝায়, আর শেষে interview-তে defend করার জন্য প্রশ্ন-উত্তর দেয়।

**Repository:** [github.com/muhammadMilon/SkillVoyager.AI-main](https://github.com/muhammadMilon/SkillVoyager.AI-main)

## পড়ার অর্ডার (simple → complex)

| # | ফিচার (ফাইলে লিংক করা) | যা শিখবেন |
|---|--------------------------|-----------|
| 01 | [Firebase Authentication & Auth Context](01-firebase-auth.md) | Multi-provider auth (email/Google/GitHub), `onAuthStateChanged` observer, React Context দিয়ে global auth state |
| 02 | [Routing, Layout & Protected Routes (RBAC)](02-routing-and-protected-routes.md) | `createBrowserRouter`, nested layout + `Outlet`, `PrivateRoute` দিয়ে role-based access control |
| 03 | [User Sync & Streak Engine](03-user-sync-and-streak.md) | Firebase user → MongoDB `upsert`, দিন-ভিত্তিক streak হিসাব, `findOneAndUpdate` |
| 04 | [Multi-Step Onboarding Flow](04-onboarding-flow.md) | Wizard/stepper state, per-step validation, skill-key normalization, একই data দুই collection-এ লেখা |
| 05 | [Progress Dashboard & Skill Radar](05-progress-dashboard-and-radar.md) | Mongoose `Map` type, auto-calculated percentage, Chart.js radar দিয়ে skill visualization |
| 06 | [Gamification & Leaderboard System](06-gamification-and-leaderboard.md) | XP/points, tier, rank-change snapshot, weekly/monthly reset, sorting at scale |
| 07 | [Gemini AI Integration Layer](07-gemini-ai-integration-layer.md) | এক জায়গায় AI call, exponential backoff retry, API key rotation, prompt engineering |
| 08 | [AI Roadmap Generator (TypeScript)](08-ai-roadmap-generator.md) | AI থেকে JSON বের করা, DB-তে save, XP + Notification side-effect, client-side retry |
| 09 | [AI Quiz: Generation & Secure Evaluation](09-ai-quiz-generation-and-evaluation.md) | AI দিয়ে quiz বানানো, correct answer লুকিয়ে রাখা, server-side scoring, skill boost |
| 10 | [Stripe Course Payments & Enrollment](10-stripe-payments-and-enrollment.md) | Stripe Checkout Session, payment verify, idempotency দিয়ে duplicate enrollment ঠেকানো |

## Tech Stack

**Frontend:** React 19 · Vite 7 · React Router 7 · Firebase Auth 12 · Tailwind CSS 4 · Framer Motion · Chart.js (react-chartjs-2) · Axios · react-toastify · @react-pdf/renderer · @stripe/stripe-js

**Backend:** Node.js · Express 5 · Mongoose 9 (MongoDB Atlas) · @google/generative-ai (Gemini) · Stripe · Nodemailer · dotenv · CORS

**Infra/Deploy:** Vercel (frontend ও backend আলাদা deploy) · MongoDB Atlas · Firebase project · Gemini API keys

---

> **টিপ (কীভাবে সবচেয়ে বেশি শিখবেন):** প্রতিটা ডকে যা লেখা আছে সেটা পড়ুন → তারপর ডকে উল্লেখ করা **আসল ফাইলটা খুলে মিলিয়ে দেখুন** → শেষে interview প্রশ্নগুলোর উত্তর **মুখে বলার চেষ্টা করুন**। যদি নিজের মুখে বলতে পারেন, তাহলেই বুঝবেন আপনি ফিচারটা সত্যিকারভাবে বুঝেছেন। শুধু পড়ে গেলে মনে হবে বুঝেছেন — বলতে গেলে ফাঁকগুলো ধরা পড়বে।
