# ০৭ — Admin Panel ও Role-Based Access Control (RBAC)

## Feature-টা আসলে কী?

App-এ দুই ধরনের user: সাধারণ **AGENT** আর **ADMIN**। Admin একটা আলাদা panel (`/admin`) পায় যেখান থেকে সে সব user, transaction, add-money request, KYC ইত্যাদি নিয়ন্ত্রণ করে। **RBAC (Role-Based Access Control)** মানে — কে কোন কাজ করতে পারবে, সেটা তার **role** দিয়ে নির্ধারিত।

এখানে মূল চ্যালেঞ্জ: নিশ্চিত করা যে একজন সাধারণ agent কোনোভাবেই admin-এর ক্ষমতা পেতে না পারে (**privilege escalation** ঠেকানো)।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| Client Guard | `src/components/admin-guard.tsx` | admin ছাড়া `/admin` UI লুকায় |
| Server Auth | `src/lib/api-helpers.ts` → `getRequestUser` | session থেকে role আনে |
| Admin verify | `src/lib/auth/admin.server.ts` | admin PIN server-side যাচাই |
| Constants | `src/lib/constants/admin.ts` | `ADMIN_PHONE`, `isAdminPhone` (client-safe) |
| Admin APIs | `src/app/api/admin/*` | সব admin-only endpoint |

### জড়িত file গুলো সংক্ষেপে

- **`src/components/admin-guard.tsx`** — client-side guard; non-admin কে `/admin` UI দেখতে দেয় না, redirect করে দেয় (শুধু UX-এর জন্য, নিরাপত্তার মূল স্তর নয়)।
- **`src/lib/api-helpers.ts → getRequestUser`** — session থেকে user ও তার DB-role নিয়ে আসে; প্রতিটা admin route এটা দিয়ে role যাচাই করে।
- **`src/lib/auth/admin.server.ts`** — server-only; admin PIN env থেকে নিয়ে constant-time compare করে; browser bundle-এ কখনো যায় না।
- **`src/lib/constants/admin.ts`** — `ADMIN_PHONE` ও `isAdminPhone` (client-safe, শুধু identity/UI gating; কোনো secret নেই)।
- **`src/app/api/admin/*`** — সব admin-only endpoint (overview, users, kyc, add-money, transactions ইত্যাদি); প্রতিটার শুরুতে `role !== "ADMIN"` চেক।

## এটা কীভাবে কাজ করে — দুই স্তরের প্রতিরক্ষা

### স্তর ১ — Client-side guard (শুধু UX-এর জন্য)

`AdminGuard` component `/admin` route-এ non-admin দের ঢুকতে দেয় না, redirect করে দেয়:

```tsx
// src/components/admin-guard.tsx
const isAdmin = authed && role === "admin" && !!phone && isAdminPhone(phone);
useEffect(() => {
  if (ready && !isAdmin) router.replace("/");
}, [ready, isAdmin, router]);
```

**গুরুত্বপূর্ণ:** এটা শুধু ভালো UX-এর জন্য (non-admin কে অপ্রয়োজনীয় page না দেখানো)। এটা আসল নিরাপত্তা **নয়** — কারণ যেকোনো client-side code bypass করা যায়।

### স্তর ২ — Server-side enforcement (আসল নিরাপত্তা)

প্রতিটা admin API নিজে server-এ role যাচাই করে:

```ts
// প্রতিটা /api/admin/* route-এর শুরুতে
const user = await getRequestUser(req);        // session cookie → DB user
if (!user || user.role !== "ADMIN") return unauthorized();
```

এখানে `role` আসে **database session** থেকে, client যা পাঠায় তা থেকে নয়। এই এক লাইনই আসল দেয়াল।

### Admin login কীভাবে আলাদা

Admin-এর PIN কখনো client bundle-এ থাকে না। এটা server-এর environment variable (`ADMIN_PIN`)-এ থাকে এবং server-side constant-time compare দিয়ে যাচাই হয়:

```ts
// src/lib/auth/admin.server.ts
export function verifyAdminCredentials(phone: string, pin: string): boolean {
  if (!isAdminPhone(phone)) return false;
  const adminPin = getAdminPin();                    // process.env.ADMIN_PIN
  if (!adminPin) return false;                       // fail closed
  return constantTimeEquals(pin, adminPin);
}
```

## কেন এই design গুরুত্বপূর্ণ (আগের bug থেকে শিক্ষা)

আগের version-এ `ADMIN_PIN = "123456"` একটা client-imported file-এ ছিল, তাই সেটা browser-এর JS bundle-এ চলে যেত — যে কেউ view-source করে admin PIN পেয়ে যেত। আর authorization হতো একটা client-controlled header (`x-user-phone`) দিয়ে — মানে যে কেউ header বদলে admin সাজতে পারত। এই দুটোই ছিল **critical vulnerability**, এখন ঠিক করা হয়েছে:

- Admin PIN → server-only env var
- Role → server-side database session থেকে

## গুরুত্বপূর্ণ concept গুলো

- **Defense in depth**: client guard + server check — দুই স্তর।
- **Never trust the client**: authorization সবসময় server-side।
- **Fail closed**: config না থাকলে (ADMIN_PIN missing) admin access বন্ধ থাকে, খোলা নয়।
- **Least privilege**: default role AGENT; ADMIN শুধু নির্দিষ্ট অনুমোদিত পথে।

---

## Interview Questions

### ১. Client-side `AdminGuard` থাকতেও প্রতিটা admin API-তে আবার server-side role check কেন দরকার?

**উত্তর:** কারণ **client-side যেকোনো কিছু bypass করা যায়**। `AdminGuard` শুধু browser-এ page/বাটন লুকায়, কিন্তু একজন attacker browser ছাড়াই সরাসরি `curl` বা Postman দিয়ে `/api/admin/users`-এ request পাঠাতে পারে — তখন React guard-এর কোনো ভূমিকাই নেই। তাই আসল দেয়াল হলো server-side check: প্রতিটা admin route নিজে session থেকে role যাচাই করে (`if role !== "ADMIN" return 401`)। Client guard কেবল UX (non-admin কে অপ্রাসঙ্গিক UI না দেখানো), আর server check হলো আসল security। এটাই **defense in depth** — এবং মূল নীতি: **never trust the client**।

### ২. Role কেন client থেকে না নিয়ে database session থেকে নেওয়া হয়?

**উত্তর:** যদি role client থেকে আসত (যেমন request header বা localStorage-এ `role: "admin"`), তাহলে যে কেউ সেই মান বদলে নিজেকে admin দাবি করতে পারত — এটাই ছিল আগের `x-user-phone` bug-এর মূল সমস্যা। এখন role আসে server-এর database session থেকে: cookie-র token → Session row → User row → `user.role`। এই chain-এর কোনো অংশ client নিয়ন্ত্রণ করতে পারে না (token httpOnly cookie, hash করে DB-তে রাখা)। ফলে role হলো **server-authoritative** — একজন agent যত কিছুই পাঠাক, server DB-তে তার role AGENT-ই থাকবে, তাই সে admin হতে পারবে না।

### ৩. "Privilege escalation" বলতে কী বোঝায়, আর এই design কীভাবে ঠেকায়?

**উত্তর:** Privilege escalation মানে কম-অধিকারসম্পন্ন একজন user কোনো ফাঁক গলে বেশি-অধিকার (যেমন admin) পেয়ে যাওয়া। আগের code-এ এটা দুইভাবে সম্ভব ছিল — (ক) admin PIN client bundle-এ থাকায় যে কেউ তা পড়ে নিতে পারত; (খ) `x-user-phone: <admin_phone>` header পাঠিয়ে যে কেউ admin সাজতে পারত। এখন ঠেকানো হয়েছে: admin PIN শুধু server env-এ, আর role আসে DB session থেকে (header trust সম্পূর্ণ বাদ)। ফলে admin হওয়ার একমাত্র পথ — সঠিক admin phone + সঠিক secret PIN দিয়ে server-এ login করা, যা agent-দের কাছে নেই।

### ৪. `verifyAdminCredentials`-এ ADMIN_PIN না থাকলে `false` return করে — একে "fail closed" বলে কেন, আর "fail open" হলে বিপদ কী?

**উত্তর:** "Fail closed" মানে — কোনো কিছু ঠিকমতো configure করা না থাকলে system **নিরাপদ দিকে** ঝুঁকবে, অর্থাৎ access **বন্ধ** রাখবে। এখানে ADMIN_PIN env-এ সেট না থাকলে `verifyAdminCredentials` সবসময় `false` দেয়, তাই admin login-ই সম্ভব হয় না। উল্টোটা "fail open" হতো যদি config না থাকলে সবাইকে ঢুকতে দিত — তখন কেউ ADMIN_PIN সেট করতে ভুলে গেলে admin panel সবার জন্য খুলে যেত, যা ভয়াবহ। Security-তে সবসময় fail closed নীতি অনুসরণ করা উচিত: সন্দেহ হলে "না" বলো।

### ৫. এই system-এ "least privilege" নীতি কীভাবে প্রতিফলিত?

**উত্তর:** Least privilege মানে প্রতিটা user-কে ঠিক ততটুকু ক্ষমতা দাও যতটুকু তার কাজের জন্য দরকার, বেশি নয়। এখানে নতুন সব user-এর default role `AGENT` (schema-তে `@default(AGENT)`), অর্থাৎ কেউ বাড়তি ক্ষমতা নিয়ে জন্মায় না। ADMIN role শুধু একটা নির্দিষ্ট, নিয়ন্ত্রিত পথে (নির্দিষ্ট phone + server-only secret PIN) পাওয়া যায়, এবং register endpoint admin number রেজিস্টার করাই আটকে দেয় (`403`)। ফলে ক্ষমতা ছড়িয়ে থাকে না, একটা সরু পথে সীমাবদ্ধ — কম attack surface, বেশি নিরাপত্তা।

### ৬. Schema-তে `SUPER_ADMIN` role আছে কিন্তু ব্যবহার নেই — role hierarchy কীভাবে বাড়ানো যায়?

**উত্তর:** বর্তমানে দুটো কার্যকর role — `AGENT` ও `ADMIN`, আর `SUPER_ADMIN` ভবিষ্যতের জন্য রাখা। Hierarchy বাড়াতে হলে সাধারণত দুটো approach: (ক) **role-ভিত্তিক** — প্রতিটা role-এর জন্য কী কী করতে পারবে তা নির্ধারণ, যেখানে SUPER_ADMIN > ADMIN > AGENT (যেমন super admin অন্য admin তৈরি/মুছতে পারবে, সাধারণ admin পারবে না); (খ) **permission-ভিত্তিক (আরও নমনীয়)** — role-এর সাথে সরাসরি না বেঁধে প্রতিটা action-কে একটা permission (`kyc.approve`, `user.delete`) হিসেবে ধরে, আর প্রতিটা role-এ permission-এর সেট map করা। বড় system-এ permission-based RBAC বেশি scalable, কারণ নতুন role/permission যোগ করতে core code বদলাতে হয় না। মূল কথা — check-টা `role === "ADMIN"`-এর মতো hardcoded না রেখে একটা `hasPermission(user, action)` abstraction-এর পেছনে নেওয়া।

### ৭. `constants/admin.ts`-কে "client-safe" বলা হয় — `ADMIN_PHONE` client bundle-এ থাকা কি ঝুঁকি নয়?

**উত্তর:** না, এটা গ্রহণযোগ্য ঝুঁকি। `ADMIN_PHONE` হলো admin-এর **identity (username-এর মতো)**, secret নয়। কেউ admin-এর phone number জেনে গেলেও কিছু করতে পারবে না, যতক্ষণ না তার কাছে secret `ADMIN_PIN` থাকে (যা শুধু server env-এ) এবং সে server-এ সফলভাবে login করে session না পায়। নিরাপত্তার মূল নীতি — **যা secret নয় তা প্রকাশ পেলে ক্ষতি নেই; আসল secret (PIN, token) কখনো client-এ যাবে না**। আগের bug ছিল ঠিক এখানেই: তখন `ADMIN_PIN` (আসল secret) client file-এ ছিল, তাই bundle-এ চলে যেত। এখন phone (identity) client-safe রাখা হয়েছে UI gating-এর জন্য, আর PIN (secret) সম্পূর্ণ server-side — এই বিভাজনটাই সঠিক।

### ৮. প্রতিটা admin action-এ audit log কেন রাখা হয়?

**উত্তর:** কারণ admin-দের হাতে সবচেয়ে বেশি ক্ষমতা — তারা টাকা approve করে, KYC পাস করায়, user নিয়ন্ত্রণ করে। এত ক্ষমতার সাথে **accountability** থাকা অপরিহার্য। প্রতিটা admin action-এ audit log (কে, কখন, কোন resource-এ, কী করল) রাখলে — (ক) কোনো ভুল/জালিয়াতি হলে তদন্ত করা যায়; (খ) কোনো admin account hijack হলে সে কী কী করেছে তা ধরা যায়; (গ) regulatory audit-এ প্রমাণ দেখানো যায়; (ঘ) internal fraud (rogue admin) নিরুৎসাহিত হয়, কারণ সব কাজ রেকর্ডে থাকে। উচ্চ-ক্ষমতাসম্পন্ন operation সবসময় auditable হওয়া উচিত — এটা security ও compliance দুটোরই দাবি।

### ৯. Non-admin logged-in user admin API-তে এলে `401 Unauthorized` না `403 Forbidden` — কোনটা সঠিক?

**উত্তর:** Semantically **403 Forbidden** বেশি সঠিক এই ক্ষেত্রে। পার্থক্যটা হলো — `401 Unauthorized` মানে "তুমি কে তা জানি না / তুমি authenticated নও" (login করো), আর `403 Forbidden` মানে "তুমি কে তা জানি, কিন্তু তোমার এই কাজের অনুমতি নেই"। একজন logged-in agent যখন admin endpoint-এ আসে, সে তো authenticated (আমরা জানি সে কে), শুধু তার permission নেই — তাই 403 উপযুক্ত। এই project-এ কিছু route সরলতার জন্য দুটো ক্ষেত্রেই `unauthorized()` (401) দেয়, যা কাজ করে কিন্তু নিখুঁত নয়। ভালো practice — auth নেই → 401, auth আছে কিন্তু permission নেই → 403; এতে client পরিষ্কার বোঝে সমস্যা কোথায় (আবার login করবে, নাকি access-ই নেই)।

### ১০. একজন admin-এর session hijack হলে ক্ষতির পরিধি (blast radius) কতটা, আর তা কমানোর উপায়?

**উত্তর:** Admin session hijack হলে blast radius বিশাল — attacker সব user দেখতে, add-money approve করতে, KYC পাস করাতে, transaction নিয়ন্ত্রণ করতে পারে, অর্থাৎ কার্যত পুরো system-এর নিয়ন্ত্রণ। এটা কমানোর উপায়: (ক) **short admin session + re-authentication** — সংবেদনশীল action (টাকা approve)-এর আগে আবার PIN চাওয়া (step-up auth); (খ) **2FA/MFA** admin-এর জন্য বাধ্যতামূলক; (গ) **IP allow-list** বা device binding — শুধু পরিচিত জায়গা থেকে admin login; (ঘ) **audit log + alerting** — অস্বাভাবিক admin কার্যকলাপে সাথে সাথে সতর্কতা; (ঙ) **least privilege / maker-checker** — একজন admin একাই সব করতে না পারা, বড় সিদ্ধান্তে দ্বিতীয় অনুমোদন। মূল দর্শন — সবচেয়ে ক্ষমতাশালী account-কেই সবচেয়ে বেশি সুরক্ষা ও নজরদারিতে রাখা।
