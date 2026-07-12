# ১০ — Routing, Middleware ও Security Headers (Next.js App Router)

## Feature-টা আসলে কী?

Next.js-এর **App Router** দিয়ে পুরো navigation structure তৈরি — কোন URL কোন page দেখাবে, কোন page login ছাড়া দেখা যাবে না, landing page কী হবে ইত্যাদি। পাশাপাশি **Middleware** দিয়ে প্রতিটা response-এ security header বসানো হয়, যা পুরো app-কে সাধারণ web attack থেকে রক্ষা করে।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| Routing | `src/app/` folder structure | file-based routing |
| Route group | `src/app/(app)/` | logged-in user-দের section |
| Landing | `src/app/page.tsx` | `/` — public landing |
| Guards | `src/components/auth-guard.tsx`, `admin-guard.tsx` | protected route পাহারা |
| Middleware | `src/middleware.ts` | security headers |
| Config | `next.config.ts` | standalone output, poweredByHeader off |

### জড়িত file গুলো সংক্ষেপে

- **`src/app/` folder structure** — Next.js App Router-এ folder/file-ই URL; প্রতিটা `page.tsx` একটা route, `route.ts` একটা API endpoint।
- **`src/app/(app)/` route group** — parenthesis-এ থাকায় URL-এ আসে না; এর ভেতরের সব page একটা shared layout (AuthGuard + AppShell) পায়।
- **`src/app/page.tsx`** — `/` public landing page (login/signup-এ ঢোকার প্রবেশদ্বার)।
- **`src/components/auth-guard.tsx`** — logged-in না হলে protected page থেকে redirect করে; hydration-এর জন্য অপেক্ষা করে।
- **`src/components/admin-guard.tsx`** — admin ছাড়া `/admin` থেকে redirect করে।
- **`src/middleware.ts`** — প্রতিটা response-এ security header (CSP, HSTS, X-Frame-Options ইত্যাদি) বসায়, page render-এর আগে চলে।
- **`next.config.ts`** — `output: "standalone"` (Docker-এর জন্য), `poweredByHeader: false` (server তথ্য না ফাঁস), strict mode।

## এটা কীভাবে কাজ করে

### ধাপ ১ — File-based Routing

App Router-এ folder structure-ই URL structure। `page.tsx` মানে একটা route:

```
src/app/page.tsx              →  /            (landing)
src/app/home/page.tsx         →  /home        (→ / এ redirect)
src/app/(app)/dashboard/…     →  /dashboard   (login লাগে)
src/app/admin/page.tsx        →  /admin       (admin লাগে)
src/app/api/recharge/route.ts →  /api/recharge (API)
```

### ধাপ ২ — Route Group `(app)`

`(app)` folder-টা parenthesis-এ, তাই এটা URL-এ আসে না — এটা শুধু একটা **grouping**। এর সুবিধা: এই group-এর সব route একটা shared layout পায় (`(app)/layout.tsx`), যেটা `AuthGuard` + `AppShell` (sidebar/header) দিয়ে মোড়ানো:

```tsx
// src/app/(app)/layout.tsx
export default function AppLayout({ children }) {
  return (
    <AuthGuard>
      <AppShell>{children}</AppShell>
    </AuthGuard>
  );
}
```

অর্থাৎ `(app)`-এর ভেতরের প্রতিটা page স্বয়ংক্রিয়ভাবে login-protected এবং একই navigation shell পায় — প্রতিটাতে আলাদা করে লিখতে হয় না।

### ধাপ ৩ — Client-side Guards

`AuthGuard` hydration শেষ হওয়ার জন্য অপেক্ষা করে, তারপর user logged-in না হলে redirect করে দেয়:

```tsx
if (!authed) {
  router.replace(pin && phone ? "/auth/pin-login" : "/");
  return;
}
if (!isSessionValid()) { /* session expired → lock + redirect */ }
```

### ধাপ ৪ — Middleware দিয়ে Security Headers

`src/middleware.ts` প্রতিটা response-এ কিছু নিরাপত্তা header বসায়:

```ts
res.headers.set("Content-Security-Policy", buildCsp());
res.headers.set("X-Content-Type-Options", "nosniff");
res.headers.set("X-Frame-Options", "DENY");
res.headers.set("Referrer-Policy", "strict-origin-when-cross-origin");
res.headers.set("Permissions-Policy", "camera=(), microphone=(), geolocation=()");
if (isProd) res.headers.set("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload");
```

এই header-গুলোর কাজ:
- **CSP** — কোন source থেকে script/style/image load হতে পারবে তা সীমিত করে (XSS-এর প্রভাব কমায়)।
- **X-Frame-Options: DENY** — অন্য কোনো site iframe-এ এই app বসাতে পারবে না (**clickjacking** ঠেকায়)।
- **X-Content-Type-Options: nosniff** — browser-কে content-type অনুমান করতে বাধা দেয়।
- **HSTS** — browser-কে সবসময় HTTPS ব্যবহার করতে বাধ্য করে।

### ধাপ ৫ — Middleware কোথায় চলে?

Middleware প্রতিটা matching request-এ, page/API render হওয়ার **আগে** চলে (Next.js-এর edge runtime-এ)। তাই এটা security header বসানোর নিখুঁত জায়গা — page, API, error — সব response একই ভাবে protected হয়।

## গুরুত্বপূর্ণ concept গুলো

- **File-based routing**: folder = URL structure।
- **Route group `(...)`**: URL না বদলে shared layout দেওয়া।
- **Middleware**: request/response-এর মাঝখানে চলা cross-cutting logic।
- **Security headers**: browser-level প্রতিরক্ষার স্তর।

---

## Interview Questions

### ১. Next.js-এর App Router আর পুরনো Pages Router-এর মূল পার্থক্য কী?

**উত্তর:** দুটোই file-based routing, কিন্তু App Router (Next 13+) অনেক বেশি শক্তিশালী। মূল পার্থক্য: (ক) App Router-এ default-ভাবে সব component **Server Component** — server-এ render হয়, browser-এ কম JS যায়, performance ভালো; client interactivity লাগলে `"use client"` দিতে হয়। (খ) App Router-এ **nested layout** সহজ — folder-ভিত্তিক shared layout (যেমন `(app)/layout.tsx`)। (গ) **Route group**, **loading/error boundary**, **streaming** — এসব built-in। Pages Router-এ সব page default-ভাবে client-side এবং layout share করা কঠিন ছিল। এই project App Router ব্যবহার করে, তাই `(app)`-এর মতো group + shared guard/layout সহজে করা গেছে।

### ২. `(app)` route group কী কাজে লাগে, এটা URL-এ প্রভাব ফেলে না কেন?

**উত্তর:** Parenthesis-এ মোড়ানো folder (যেমন `(app)`) হলো **route group** — এটা শুধু code সংগঠন ও shared layout দেওয়ার জন্য, URL path-এ যোগ হয় না (`/(app)/dashboard` নয়, শুধু `/dashboard`)। এর মূল সুবিধা: এই group-এর সব page একটা common layout ভাগ করে নিতে পারে। এখানে `(app)/layout.tsx`-এ `AuthGuard` + `AppShell` মোড়ানো, তাই এর ভেতরের প্রতিটা page স্বয়ংক্রিয়ভাবে login-protected এবং একই sidebar/header পায় — প্রতিটা page-এ আলাদা করে guard বা layout লেখার দরকার নেই। এটা DRY (Don't Repeat Yourself) নীতির চমৎকার প্রয়োগ।

### ৩. Client-side guard (`AuthGuard`) থাকলেও কেন সেটা যথেষ্ট নিরাপত্তা নয়?

**উত্তর:** `AuthGuard` শুধু browser-এ page দেখানো/লুকানো নিয়ন্ত্রণ করে — এটা UX-এর জন্য (logged-out user কে protected page না দেখানো)। কিন্তু এটা bypass করা যায়: কেউ JS নিষ্ক্রিয় করে বা সরাসরি API-তে request পাঠিয়ে data চাইতে পারে। তাই আসল নিরাপত্তা থাকে **API layer-এ** — প্রতিটা protected API `getRequestUser` দিয়ে session যাচাই করে, না থাকলে 401 দেয়। অর্থাৎ page দেখা আটকানো এক ব্যাপার, আর data access আটকানো আরেক ব্যাপার; দ্বিতীয়টা সবসময় server-side হতে হবে। Client guard আর server check মিলে **defense in depth**।

### ৪. `X-Frame-Options: DENY` আর CSP header কোন কোন attack থেকে রক্ষা করে?

**উত্তর:** **`X-Frame-Options: DENY`** (বা CSP-র `frame-ancestors 'none'`) ঠেকায় **clickjacking** — যেখানে attacker একটা invisible iframe-এ তোমার app বসিয়ে user-কে ধোঁকা দিয়ে অপ্রত্যাশিত জায়গায় click করায় (যেমন "টাকা পাঠান" বাটনে)। DENY দিলে কোনো site-ই এই app iframe-এ বসাতে পারে না। **CSP (Content-Security-Policy)** ঠেকায় মূলত **XSS** — এটা নির্ধারণ করে কোন উৎস থেকে script/style/image চলতে পারবে (`default-src 'self'`)। ফলে কোনো malicious script কোনোভাবে ঢুকলেও external server-এ data পাঠানো বা বাইরের script load করা অনেক কঠিন হয়ে যায়। এগুলো browser-level প্রতিরক্ষা — server code-এর বাইরে বাড়তি একটা সুরক্ষাস্তর।

### ৫. Security header বসানোর জন্য middleware-ই সেরা জায়গা কেন?

**উত্তর:** কারণ middleware প্রতিটা matching request-এ, response তৈরি হওয়ার **আগে**, একটা কেন্দ্রীয় জায়গায় চলে। যদি প্রতিটা page/API-তে আলাদা করে header বসাতাম, তাহলে কোথাও ভুলে যাওয়ার ঝুঁকি থাকত এবং code duplicate হতো। Middleware-এ একবার লিখলে সেটা **সব response** (page, API, এমনকি error page)-এ সমানভাবে প্রযোজ্য — এটা একটা **cross-cutting concern**-এর আদর্শ সমাধান। এছাড়া middleware routing-এর আগে চলে বলে চাইলে এখানে redirect/auth-এর মতো কাজও করা যায়। তাই security header-এর মতো "সবার জন্য একই নিয়ম" middleware-এ রাখাই সবচেয়ে নির্ভরযোগ্য ও maintainable।

### ৬. Middleware-এ `matcher` config দিয়ে কিছু path বাদ দেওয়া হয় কেন?

**উত্তর:** `matcher` নির্ধারণ করে middleware কোন কোন request-এ চলবে। এখানে static asset (`_next/static`, image, favicon, `.png/.svg` ইত্যাদি) বাদ দেওয়া হয়েছে দুই কারণে: (ক) **performance** — প্রতিটা ছবি/CSS/JS file-এর জন্য middleware চালানো অপ্রয়োজনীয় overhead, কারণ এগুলো তো dynamic response নয়; (খ) static asset-এ security header (CSP ইত্যাদি) বসানোর দরকার নেই, ওগুলো HTML/API-র জন্য প্রাসঙ্গিক। তাই matcher দিয়ে middleware-কে শুধু meaningful request (page, API)-এ সীমাবদ্ধ রাখা হয়, যাতে অকারণে হাজারো asset request slow না হয়। এটা একটা সাধারণ optimization — cross-cutting logic শুধু সেখানেই চালানো যেখানে দরকার।

### ৭. CSP-তে `'unsafe-inline'`/`'unsafe-eval'` রাখা হয়েছে — এটা কি CSP-কে দুর্বল করে না, উন্নত করার উপায় কী?

**উত্তর:** হ্যাঁ, `'unsafe-inline'` (inline script/style অনুমতি) ও `'unsafe-eval'` CSP-কে দুর্বল করে, কারণ এগুলো থাকলে inline injected script চলতে পারে — যা CSP-র মূল উদ্দেশ্য (XSS ঠেকানো) কিছুটা লঘু করে। এগুলো রাখতে হয়েছে বাস্তব কারণে: Next.js hydration-এর জন্য inline bootstrap script বসায়, framer-motion inline style দেয়, আর dev mode-এ HMR-এর জন্য `eval` লাগে। উন্নত করার সঠিক উপায় — **nonce-based CSP**: প্রতিটা request-এ একটা random nonce তৈরি করে বৈধ script-গুলোতে সেটা বসানো, আর CSP-তে `'unsafe-inline'`-এর বদলে `'nonce-xxx'` অনুমতি দেওয়া; তাহলে শুধু সঠিক nonce-ওয়ালা script চলবে, attacker-এর injected script নয়। এই code-এ production-এ `'unsafe-eval'` বাদ দেওয়া হয়েছে (শুধু dev-এ), যা একটা ভালো মাঝপথ; পূর্ণ কড়া করতে nonce pipeline যোগ করা যায়।

### ৮. `next.config.ts`-এ `output: "standalone"` কী করে, Docker-এ এটা কেন গুরুত্বপূর্ণ?

**উত্তর:** `output: "standalone"` Next.js-কে বলে build-এর সময় একটা **স্বয়ংসম্পূর্ণ (self-contained) minimal server bundle** তৈরি করতে — যেখানে শুধু runtime-এ যা যা দরকার (প্রয়োজনীয় node_modules সহ একটা `server.js`) তাই থাকে, পুরো `node_modules` নয়। Docker-এ এটা গুরুত্বপূর্ণ কারণ এতে final image অনেক ছোট হয় (কয়েকশো MB বাঁচে), যা দ্রুত deploy, কম storage, ও কম attack surface দেয়। এই project-এর Dockerfile ঠিক এই standalone output কপি করে একটা slim runner image বানায় (`node server.js`)। Standalone ছাড়া পুরো dev dependencies সহ image bloated হতো। সংক্ষেপে — standalone = production-এর জন্য অপ্টিমাইজড, ছোট, portable server।

### ৯. Next.js App Router-এ Server Component আর Client Component-এর পার্থক্য কী, `"use client"` কখন লাগে?

**উত্তর:** App Router-এ প্রতিটা component **default-ভাবে Server Component** — এগুলো শুধু server-এ render হয়, browser-এ এদের JS যায় না (তাই bundle ছোট, data fetching server-এ সহজ, secret নিরাপদ)। কিন্তু Server Component-এ browser-only জিনিস (state, effect, event handler, `window`, hooks) ব্যবহার করা যায় না। যেখানে interactivity লাগে — `useState`, `useEffect`, onClick, বা zustand store — সেই component-এর উপরে `"use client"` দিতে হয়, যা সেটাকে Client Component বানায় (browser-এ hydrate হয়)। এই project-এ interactive page/component (form, guard, store ব্যবহারকারী) গুলোতে `"use client"` আছে। নিয়ম — যতটা সম্ভব Server Component রাখা (performance), আর শুধু যেখানে browser interactivity লাগে সেখানে `"use client"` দেওয়া, সাধারণত leaf/interactive অংশে।

### ১০. Middleware edge runtime-এ চলে — তাই এখানে Prisma/DB query করা যায় না কেন?

**উত্তর:** Next.js middleware default-ভাবে **Edge runtime**-এ চলে — এটা একটা হালকা, দ্রুত পরিবেশ (V8-based) যা full Node.js নয়। Edge-এ Node-এর অনেক API (TCP socket, native module) নেই, আর Prisma-র সাধারণ engine একটা native binary যা DB-তে সরাসরি TCP connection করে — তাই সেটা edge middleware-এ চলে না। এছাড়া middleware প্রতিটা request-এ চলে, তাই সেখানে DB query করলে বড় latency ও connection-pool চাপ তৈরি হতো। এই কারণেই এই project-এ middleware শুধু security header বসায় (কোনো DB/auth lookup নয়), আর আসল auth যাচাই (`getSessionUser`, যা DB ছোঁয়) হয় route handler-এ, যা full Node.js runtime-এ চলে। নীতি — middleware-এ হালকা, DB-বিহীন cross-cutting কাজ রাখা; ভারী/DB-নির্ভর logic route handler-এ।
