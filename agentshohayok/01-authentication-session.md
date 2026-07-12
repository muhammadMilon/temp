# ০১ — Authentication ও Session Management

## Feature-টা আসলে কী?

যেকোনো fintech app-এর সবচেয়ে গুরুত্বপূর্ণ অংশ হলো **কে user, আর সে আসলেই সে কিনা** — এটা নিশ্চিত করা। এই feature-টা তিনটা প্রশ্নের উত্তর দেয়:

1. **Authentication** — তুমি কে? (login/register)
2. **Session** — একবার login করার পর প্রতিবার phone/PIN ছাড়াই তোমাকে কীভাবে চেনা যাবে?
3. **Authorization** — তুমি কি এই কাজটা করার অনুমতি রাখো? (agent vs admin)

এই project-এ user একটা **phone number + 5-digit PIN** দিয়ে login করে। Admin login করে phone + একটা secret PIN দিয়ে যেটা শুধু server-এ থাকে।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| API | `src/app/api/auth/login/route.ts` | PIN verify করে session বানায় |
| API | `src/app/api/auth/register/route.ts` | নতুন account বানায় (auto-login) |
| API | `src/app/api/auth/logout/route.ts` | session মুছে দেয় |
| API | `src/app/api/auth/check/route.ts` | phone আগে registered কিনা দেখে |
| Core | `src/lib/auth/session.ts` | session তৈরি/যাচাই/মুছে ফেলা |
| Core | `src/lib/auth/password.ts` | PIN hash ও verify (scrypt) |
| Core | `src/lib/auth/admin.server.ts` | admin PIN server-side verify |
| Helper | `src/lib/api-helpers.ts` | প্রতি request-এ user কে বের করে |
| DB | `prisma/schema.prisma` → `Session` model | session DB-তে রাখে |

### জড়িত file গুলো সংক্ষেপে

- **`src/app/api/auth/login/route.ts`** — user-এর phone+PIN নেয়, scrypt hash যাচাই করে; সঠিক হলে একটা Session বানিয়ে httpOnly cookie সেট করে। Admin হলে env-এর `ADMIN_PIN` দিয়ে যাচাই করে।
- **`src/app/api/auth/register/route.ts`** — নতুন account তৈরি করে (PIN hash করে রাখে), আগে থেকে থাকা কোনো account-এর PIN কখনো overwrite করে না; সফল হলে সাথে সাথে auto-login (session) দিয়ে দেয়।
- **`src/app/api/auth/logout/route.ts`** — DB থেকে সংশ্লিষ্ট Session row মুছে দেয় এবং cookie clear করে, যাতে session সঙ্গে সঙ্গে অকার্যকর হয়।
- **`src/app/api/auth/check/route.ts`** — একটা phone আগে registered কিনা জানায়, যাতে client ঠিক করতে পারে পরের screen হবে `pin-login` (পুরনো user) নাকি `pin-create` (নতুন user)।
- **`src/lib/auth/session.ts`** — session-এর মূল ইঞ্জিন: random token বানানো, তার SHA-256 hash DB-তে রাখা, cookie-র flag ঠিক করা, আর প্রতি request-এ token থেকে user বের করা (`getSessionUser`)।
- **`src/lib/auth/password.ts`** — scrypt দিয়ে PIN hash (`hashPin`) ও constant-time verify (`verifyPin`); পুরনো plaintext PIN চেনার helper-ও আছে।
- **`src/lib/auth/admin.server.ts`** — server-only file, admin PIN env থেকে নিয়ে constant-time compare করে যাচাই করে; কখনো browser bundle-এ যায় না।
- **`src/lib/api-helpers.ts`** — `getRequestUser(req)` দিয়ে প্রতিটা protected route session থেকে user পায়; সাথে `unauthorized()`, `badRequest()`-এর মতো response helper।
- **`prisma/schema.prisma → Session`** — session-এর `tokenHash`, `expiresAt`, `ipAddress`, `userAgent` DB-তে রাখে; এটাই stateful session-এর ভিত্তি।

## পুরো flow-টা কীভাবে কাজ করে

### ধাপ ১ — PIN কখনো plaintext-এ রাখা হয় না

User-এর PIN সরাসরি database-এ রাখলে, database leak হলেই সব PIN বেরিয়ে যাবে। তাই আমরা **scrypt** নামের একটা one-way hashing algorithm ব্যবহার করি। প্রতি user-এর জন্য আলাদা random **salt** যোগ করা হয়।

```ts
// src/lib/auth/password.ts
export function hashPin(pin: string): string {
  const salt = randomBytes(16).toString("hex");
  const hash = scryptSync(pin, salt, KEYLEN, { N: COST }).toString("hex");
  return `scrypt$${COST}$${salt}$${hash}`;
}
```

Verify করার সময় submitted PIN আবার একই salt দিয়ে hash করা হয় এবং **constant-time compare** (`timingSafeEqual`) দিয়ে মেলানো হয় — যাতে timing attack দিয়ে PIN গেস করা না যায়।

### ধাপ ২ — Login করলে server একটা Session বানায়

সঠিক PIN দিলে server একটা **256-bit random token** বানায়। এই token-টা browser-এ যায়, কিন্তু database-এ শুধু তার **SHA-256 hash** রাখা হয়।

```ts
// src/lib/auth/session.ts
const token = randomBytes(32).toString("base64url");
await db.session.create({
  data: { userId, tokenHash: hashToken(token), expiresAt, ipAddress, userAgent },
});
```

কারণ: database leak হলেও শুধু hash পাওয়া যাবে, আসল token নয়, তাই সেই hash দিয়ে কেউ session জাল করতে পারবে না।

### ধাপ ৩ — Token একটা httpOnly cookie-তে যায়

```ts
res.cookies.set(SESSION_COOKIE, token, {
  httpOnly: true,                                    // JavaScript পড়তে পারবে না
  secure: process.env.NODE_ENV === "production",     // শুধু HTTPS-এ যাবে
  sameSite: "lax",                                   // CSRF থেকে কিছুটা রক্ষা
  path: "/",
  expires: expiresAt,
});
```

`httpOnly: true` মানে এই cookie-টা browser-এর `document.cookie` দিয়ে পড়া যায় না — তাই কোনো XSS script token চুরি করতে পারবে না।

### ধাপ ৪ — প্রতিটা পরের request-এ cookie থেকে user বের করা হয়

Browser same-origin request-এ cookie automatically পাঠায়। Server সেই cookie-এর token নিয়ে hash করে, database-এ session খুঁজে, তারপর **user ও role database থেকে** নিয়ে আসে।

```ts
// src/lib/api-helpers.ts — সব protected route এটা ব্যবহার করে
export async function getRequestUser(req: NextRequest) {
  return getSessionUser(req);   // cookie → session → DB user
}
```

এখানে সবচেয়ে গুরুত্বপূর্ণ নিরাপত্তা নীতি: **identity ও role সবসময় server-এর database থেকে আসে, client যা পাঠায় তা থেকে নয়।**

### ধাপ ৫ — Brute-force ঠেকাতে Rate Limiting

5-digit PIN মানে মাত্র ১,০০,০০০ combination — bot দিয়ে সহজেই try করা যায়। তাই login/register endpoint-এ একটা sliding-window rate limiter আছে (`src/lib/security/rate-limit.ts`) — একই IP+phone থেকে 15 মিনিটে সর্বোচ্চ 10 বার।

## গুরুত্বপূর্ণ concept গুলো

- **Hashing vs Encryption**: Hashing one-way (ফেরানো যায় না), password/PIN-এর জন্য এটাই লাগে।
- **Stateful session**: session-এর তথ্য server (DB)-তে থাকে, তাই logout করলে সাথে সাথে সেটা invalidate করা যায়।
- **Defense in depth**: hashing + rate limiting + httpOnly cookie — একটা layer ভাঙলেও অন্যটা রক্ষা করে।

---

## Interview Questions

### ১. `httpOnly` cookie কেন ব্যবহার করা হয়, আর এটা না দিলে কী ঝুঁকি?

**উত্তর:** `httpOnly` cookie browser-এর JavaScript (`document.cookie`) দিয়ে পড়া বা পরিবর্তন করা যায় না — শুধু browser নিজে server-এ পাঠায়। এর সবচেয়ে বড় সুবিধা হলো **XSS (Cross-Site Scripting) attack থেকে রক্ষা**: যদি attacker কোনোভাবে page-এ malicious script ঢোকায়, সেই script session token পড়ে চুরি করতে পারবে না। `httpOnly` না দিলে token localStorage বা সাধারণ cookie-তে থাকলে একটা XSS-ই পুরো account hijack করার জন্য যথেষ্ট। এজন্য session token কখনো localStorage-এ রাখা উচিত নয়।

### ২. PIN hash করতে `bcrypt` না হয়ে `scrypt` কেন? আর plaintext-এ রাখলে সমস্যা কী?

**উত্তর:** Plaintext-এ রাখলে database leak হলেই সব user-এর PIN attacker-এর হাতে চলে যায় (এবং মানুষ একই PIN অন্য জায়গায়ও ব্যবহার করে)। তাই one-way hash দরকার। `scrypt` বেছে নেওয়া হয়েছে দুই কারণে — (ক) এটা **memory-hard**, অর্থাৎ GPU দিয়ে দ্রুত brute-force করা কঠিন ও ব্যয়বহুল; (খ) এটা Node.js-এর built-in `crypto` module-এ আছে, তাই কোনো extra dependency লাগে না। `bcrypt`-ও ঠিক আছে, কিন্তু সেটা external package। মূল কথা — একটা **slow, salted, memory-hard** hash হলেই চলবে; কখনো plain `md5`/`sha256` PIN-এর জন্য ব্যবহার করা যাবে না, কারণ সেগুলো দ্রুত এবং সহজে rainbow-table দিয়ে ভাঙা যায়।

### ৩. Session token database-এ hash করে রাখা হয় কেন, সরাসরি token রাখলে সমস্যা কোথায়?

**উত্তর:** যদি আসল token সরাসরি database-এ রাখা হতো, তাহলে database leak হলে attacker প্রতিটা token নিয়ে সোজা cookie-তে বসিয়ে যেকোনো user সাজতে পারত। কিন্তু আমরা শুধু token-এর **SHA-256 hash** রাখি। Login-এর সময় user-এর কাছে থাকা আসল token আবার hash করে DB-র hash-এর সাথে মেলানো হয়। ফলে DB leak হলেও attacker শুধু hash পাবে — আর hash থেকে আসল token বের করা (preimage) কার্যত অসম্ভব। এটা ঠিক PIN hashing-এর মতোই নীতি, শুধু এখানে session token-এর ক্ষেত্রে প্রয়োগ।

### ৪. Stateful session (DB-তে) আর stateless JWT-এর মধ্যে পার্থক্য কী, আর এখানে কোনটা বেছে নেওয়া হয়েছে কেন?

**উত্তর:** **JWT (stateless)**-এ user-এর তথ্য একটা signed token-এর ভেতরেই থাকে; server প্রতি request-এ DB না ছুঁয়ে শুধু signature verify করে। সুবিধা: দ্রুত, scalable। অসুবিধা: token একবার issue হলে **expire না হওয়া পর্যন্ত সেটা বাতিল করা কঠিন** — কেউ logout করলে বা account ban করলেও token কাজ করতে থাকে। **Stateful session (এই project)**-এ session-এর তথ্য DB-তে থাকে, তাই logout/ban করলে সাথে সাথে row মুছে দিলেই session অকেজো। একটা fintech app-এ instant revocation দরকারি (fraud/লগআউট), তাই এখানে DB-backed session বেছে নেওয়া হয়েছে। Trade-off হলো প্রতি request-এ একটা DB lookup।

### ৫. Login endpoint-এ rate limiting না থাকলে কী হতে পারত?

**উত্তর:** PIN মাত্র 5 digit, অর্থাৎ সর্বোচ্চ ১,০০,০০০টা সম্ভাব্য মান। Rate limiting না থাকলে একটা automated script সেকেন্ডে হাজারো request পাঠিয়ে কয়েক মিনিটেই সব combination try করে যেকোনো account-এর PIN বের করে ফেলতে পারত (brute-force attack)। Hashing এই আক্রমণ ঠেকায় না, কারণ attacker তো live endpoint-এ guess করছে। তাই **rate limiting + lockout** এখানে মূল প্রতিরক্ষা — এটা একই IP/phone থেকে অল্প সময়ে অনেক ব্যর্থ চেষ্টা ব্লক করে দেয়। এজন্য বলা হয় hashing আর rate limiting দুটোই একসাথে দরকার, একটা আরেকটার বিকল্প নয়।

### ৬. Cookie-তে `SameSite: "lax"` দেওয়া হয়েছে — এটা কী করে, আর কোন attack ঠেকায়?

**উত্তর:** `SameSite` নিয়ন্ত্রণ করে অন্য site থেকে আসা request-এ browser এই cookie পাঠাবে কিনা। `lax` মানে — cookie শুধু same-site request আর top-level navigation (link-এ click)-এ যাবে, কিন্তু অন্য site-এর ভেতর থেকে করা background request (যেমন malicious site-এর একটা লুকানো form/image)-এ যাবে না। এটা মূলত **CSRF (Cross-Site Request Forgery)** ঠেকায় — যেখানে attacker তোমাকে দিয়ে অজান্তেই তোমার logged-in session ব্যবহার করে টাকা transfer-এর মতো request পাঠায়। `strict` আরও কড়া কিন্তু UX খারাপ করে (বাইরের link থেকে এলে logged-out দেখায়), তাই `lax` হলো নিরাপত্তা ও ব্যবহারযোগ্যতার সুন্দর ভারসাম্য।

### ৭. Session expiry কীভাবে handle হয়, আর একটা expired session দিয়ে request এলে কী ঘটে?

**উত্তর:** প্রতিটা Session-এ একটা `expiresAt` (7 দিন পরের সময়) রাখা হয় তৈরির সময়। `getSessionUser`-এ token দিয়ে session খুঁজে পাওয়ার পর চেক করা হয় `session.expiresAt.getTime() < Date.now()` — অর্থাৎ মেয়াদ শেষ কিনা। শেষ হলে `null` return করা হয়, ফলে route `unauthorized()` (401) দেয় এবং user-কে আবার login করতে হয়। এই design-এর সুবিধা — session চিরকাল valid থাকে না, চুরি হওয়া cookie-ও সীমিত সময়ের বেশি কাজে লাগে না। আরও শক্ত করতে চাইলে expired session-গুলো একটা background job দিয়ে DB থেকে পরিষ্কার (cleanup) করা যায়।

### ৮. `getSessionUser` প্রতি request-এ DB query করে — এতে performance-এ কী প্রভাব, আর কীভাবে optimize করা যায়?

**উত্তর:** হ্যাঁ, stateful session-এর trade-off হলো প্রতিটা authenticated request-এ অন্তত একটা DB lookup (`Session` → `User`)। কম-মাঝারি traffic-এ এটা নগণ্য, বিশেষত `tokenHash`-এ index থাকায় lookup দ্রুত। কিন্তু বড় scale-এ optimize করা যায়: (ক) session data একটা fast in-memory store (Redis)-এ cache করা, যাতে প্রতি request-এ Postgres না ছুঁতে হয়; (খ) `lastSeenAt` update-টা "best-effort" ও fire-and-forget রাখা (এই code-এ তা-ই করা আছে), যাতে সেটা request block না করে; (গ) খুব high-scale-এ hybrid approach — short-lived signed token + মাঝে মাঝে DB revalidation। মূল কথা — নিরাপত্তা (instant revocation) আর performance-এর মধ্যে সচেতন ভারসাম্য।

### ৯. Register-কে "create-only" করা হয়েছে (আগের account overwrite করে না) — এটা কোন attack ঠেকায়?

**উত্তর:** এটা **account takeover** ঠেকায়। আগের code-এ register একটা `upsert` করত — মানে কেউ যদি অন্য কারো (এমনকি নিজের নয় এমন) phone number দিয়ে register করত, তাহলে সেই বিদ্যমান account-এর PIN নতুন PIN দিয়ে বদলে যেত, ফলে আসল মালিক লক-আউট হয়ে যেত এবং attacker account দখল করে নিত। এখন register আগে চেক করে account-এ ইতিমধ্যে একটা hashed PIN আছে কিনা; থাকলে সে `409 Conflict` দিয়ে ফিরিয়ে দেয় ("already registered, please log in")। ফলে কেবল নতুন (বা কখনো PIN সেট না-করা placeholder) account-ই তৈরি হয়, বিদ্যমান কারো account জোর করে reset করা যায় না।

### ১০. Admin login আর সাধারণ user login আলাদা path কেন, আর সফল admin login-এ DB-তে admin user `upsert` করা হয় কেন?

**উত্তর:** সাধারণ user-এর PIN DB-তে hash করে রাখা থাকে, কিন্তু admin-এর PIN রাখা হয় server-এর env variable-এ (`ADMIN_PIN`) — যাতে সেটা কখনো DB বা client bundle-এ না যায়। তাই login route দুই ভাগ: admin phone হলে env PIN দিয়ে constant-time compare, নাহলে DB-র hash দিয়ে verify। সফল admin login-এ DB-তে admin user `upsert` করা হয় কারণ Session table-এর `userId` একটা foreign key — session বানাতে হলে একটা বাস্তব User row থাকতেই হবে, এবং তার `role` অবশ্যই `ADMIN` হতে হবে (যাতে পরে API-গুলো role DB থেকে পড়ে সঠিকভাবে চিনতে পারে)। `upsert` নিশ্চিত করে admin row না থাকলে তৈরি হবে, থাকলে role ঠিক করে দেবে।
