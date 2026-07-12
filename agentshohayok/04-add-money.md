# ০৪ — Add Money (Request + Admin Approval Workflow)

## Feature-টা আসলে কী?

Agent তার wallet-এ টাকা যোগ করতে চাইলে সে bKash/Nagad/Bank দিয়ে admin-কে টাকা পাঠায়, তারপর app-এ একটা **request** জমা দেয় (কত টাকা, কোন method, external TrxID)। Admin সেই request যাচাই করে **approve/reject/hold** করে। Approve করলে তবেই agent-এর balance বাড়ে।

এটা একটা **approval workflow** — টাকা সরাসরি যোগ হয় না, একজন মানুষ (admin) যাচাই করার পর হয়। Fintech-এ এটা fraud ঠেকানোর মূল উপায়।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| User UI | `src/app/(app)/add-money/page.tsx` | request form |
| User API | `src/app/api/add-money/route.ts` | request তৈরি (POST) + list (GET) |
| Admin UI | `src/app/admin/add-money/page.tsx` | pending request-এর তালিকা |
| Admin API | `src/app/api/admin/add-money/[id]/route.ts` | approve/reject/hold |
| DB | `AddMoneyRequest`, `Wallet`, `Transaction`, `LedgerEntry`, `AuditLog` | records |

### জড়িত file গুলো সংক্ষেপে

- **`src/app/(app)/add-money/page.tsx`** — user-এর request form: কত টাকা, কোন method (bKash/Nagad/Bank), external TrxID; আর নিজের আগের request-এর তালিকা।
- **`src/app/api/add-money/route.ts`** — `POST` দিয়ে নতুন request (`PENDING`) তৈরি করে; `GET` দিয়ে শুধু নিজের request ফেরত দেয়। এখানে balance-এ কিছুই যোগ হয় না।
- **`src/app/admin/add-money/page.tsx`** — admin-এর UI: সব pending add-money request-এর তালিকা, approve/reject/hold বাটন।
- **`src/app/api/admin/add-money/[id]/route.ts`** — admin-only endpoint; approve করলে atomic-ভাবে balance বাড়ায় + Transaction + ledger + audit; reject/hold করলে শুধু status বদলায়।
- **`AddMoneyRequest` (DB)** — request-এর মূল record: amount, method, externalTrxId, status, reviewedBy।
- **`Wallet` / `Transaction` / `LedgerEntry` / `AuditLog` (DB)** — approve হলে টাকা যোগ, লেনদেন record, হিসাব ও log এখানে হয়।

## পুরো flow-টা কীভাবে কাজ করে

### ধাপ ১ — User request জমা দেয় (কোনো টাকা যোগ হয় না)

```ts
const parsed = addMoneySchema.safeParse(await req.json().catch(() => null));
if (!parsed.success) return badRequest(...);

const request = await db.addMoneyRequest.create({
  data: { trxId, userId: user.id, amount, method: dbMethod, externalTrxId: transactionId, status: "PENDING" },
});
```

লক্ষ্য করো — এখানে wallet-এর balance-এ **কিছুই যোগ হচ্ছে না**। শুধু একটা `PENDING` status-এর request তৈরি হচ্ছে। বল এখন admin-এর কোর্টে।

### ধাপ ২ — Admin approve করলে তবেই টাকা যোগ হয়

Admin API-তে প্রথমেই role যাচাই — শুধু admin এখানে ঢুকতে পারবে:

```ts
const admin = await getRequestUser(req);
if (!admin || admin.role !== "ADMIN") return unauthorized();
```

তারপর request-টা এখনো `PENDING` কিনা দেখা হয় (double-approve ঠেকাতে):

```ts
if (request.status !== "PENDING") return badRequest("Request is not in PENDING state");
```

Approve হলে একটা `$transaction`-এর ভেতরে চারটা কাজ একসাথে হয়:

```ts
await db.$transaction([
  db.addMoneyRequest.update({ where: { id }, data: { status: "SUCCESS", reviewedBy: admin.id, ... } }),
  db.wallet.update({ where: { id: wallet.id }, data: { balance: balanceAfter } }),
  db.transaction.create({ data: { trxId, type: "add_money", status: "SUCCESS", ... } }),
  db.auditLog.create({ data: { action: "ADD_MONEY_APPROVE", userId: admin.id, ... } }),
]);
```

### ধাপ ৩ — Status State Machine

একটা request-এর জীবনচক্র একটা **state machine**:

```
PENDING ──approve──▶ SUCCESS   (balance বাড়ে)
   │
   ├───reject───▶ FAILED    (কিছু হয় না)
   │
   └────hold────▶ HOLD      (আটকে রাখা, পরে সিদ্ধান্ত)
```

মূল নিয়ম: শুধু `PENDING` থেকেই অন্য state-এ যাওয়া যায়। একবার `SUCCESS`/`FAILED` হয়ে গেলে আর বদলানো যায় না — এতে একই request দুবার approve করে দুবার টাকা যোগ করা আটকানো হয়।

### ধাপ ৪ — Audit ও Accountability

প্রতিটা approve/reject-এ `reviewedBy: admin.id` এবং একটা `AuditLog` রাখা হয় — কোন admin, কখন, কোন request-এ কী সিদ্ধান্ত নিল তার স্থায়ী প্রমাণ। Fintech-এ এই accountability আইনগতভাবেও জরুরি।

## গুরুত্বপূর্ণ concept গুলো

- **Approval workflow / maker-checker**: একজন request করে (maker), আরেকজন অনুমোদন করে (checker) — fraud ঠেকানোর classic pattern।
- **State machine**: নির্দিষ্ট নিয়মে status বদলায়, যেকোনো state থেকে যেকোনো state-এ যাওয়া যায় না।
- **Server-side authorization**: টাকা যোগ করার আগে server নিজে role যাচাই করে।

---

## Interview Questions

### ১. Add money-তে user-এর balance সরাসরি না বাড়িয়ে admin approval কেন রাখা হয়েছে?

**উত্তর:** কারণ টাকাটা app-এর বাইরে (bKash/Nagad/Bank-এ) পাঠানো হয়, আর app সেটা যাচাই করতে পারে না। User যদি নিজেই নিজের balance বাড়াতে পারত, তাহলে টাকা না পাঠিয়েই মিথ্যা TrxID দিয়ে unlimited balance বানিয়ে ফেলতে পারত — সরাসরি fraud। তাই **maker-checker** pattern: user শুধু request করে (`PENDING`), আর admin আসল টাকা এসেছে কিনা যাচাই করে তবেই approve করে। এতে টাকা যোগ হওয়ার আগে একজন মানুষ verification-এর দায়িত্বে থাকে।

### ২. Request-এর status "state machine" বলতে কী বোঝায়, আর এটা কেন গুরুত্বপূর্ণ?

**উত্তর:** State machine মানে request কেবল কিছু নির্দিষ্ট নিয়মে এক অবস্থা থেকে আরেক অবস্থায় যেতে পারে — এখানে `PENDING → SUCCESS/FAILED/HOLD`, কিন্তু `SUCCESS` থেকে আর কোথাও যাওয়া যায় না। এটা গুরুত্বপূর্ণ কারণ এটা **invalid transition** আটকায়: যেমন একই request দুবার approve করা, বা reject হয়ে যাওয়া request আবার approve করা। Code-এ `if (request.status !== "PENDING") return badRequest(...)` দিয়ে এই নিয়ম প্রয়োগ করা হয়, যা double-crediting-এর মতো আর্থিক bug ঠেকায়।

### ৩. একই request দুবার approve হয়ে গেলে কী সমস্যা হতো, আর কোড কীভাবে ঠেকায়?

**উত্তর:** দুবার approve হলে agent-এর balance দুবার বাড়ত অথচ টাকা এসেছে একবার — system-এ ভুয়া টাকা তৈরি হতো। কোড এটা ঠেকায় দুইভাবে: (ক) approve করার আগে চেক করে request এখনো `PENDING` কিনা; একবার `SUCCESS` হয়ে গেলে দ্বিতীয়বার চেষ্টা `badRequest` পায়। (খ) পুরো approve কাজটা `$transaction`-এ atomic, তাই status update আর balance বৃদ্ধি একসাথে ঘটে। তবে আরও শক্ত করতে হলে status check-টাও transaction-এর ভেতরে conditional update হিসেবে রাখা ভালো, যাতে দুই admin ঠিক একই মুহূর্তে approve করলেও একটাই কার্যকর হয়।

### ৪. এখানে admin authorization কীভাবে নিশ্চিত করা হয়, শুধু frontend-এ admin page লুকিয়ে রাখলে যথেষ্ট নয় কেন?

**উত্তর:** Admin API route-এ প্রতিবার `getRequestUser(req)` দিয়ে session থেকে user বের করে `if (admin.role !== "ADMIN")` চেক করা হয় — অর্থাৎ role আসে **server-এর database session** থেকে, client যা দাবি করে তা থেকে নয়। শুধু frontend-এ admin page/বাটন লুকিয়ে রাখা যথেষ্ট নয়, কারণ frontend যেকোনো ব্যবহারকারী bypass করতে পারে — সে সরাসরি `curl` দিয়ে `/api/admin/add-money/[id]`-এ request পাঠাতে পারে। তাই আসল authorization সবসময় **server-side**-এ থাকতে হবে; frontend guard শুধু UX-এর জন্য।

### ৫. `reviewedBy` আর `AuditLog` রাখার দরকার কী?

**উত্তর:** এগুলো **accountability ও audit trail**-এর জন্য। `reviewedBy: admin.id` বলে দেয় কোন admin এই request-এ সিদ্ধান্ত নিয়েছে, আর `AuditLog` কখন কী action হয়েছে তার স্থায়ী রেকর্ড রাখে। আর্থিক system-এ পরে যদি কোনো বিতর্ক/জালিয়াতি তদন্ত হয়, তখন "কে কখন কোন টাকা approve করেছিল" — এই তথ্য অপরিহার্য। এটা শুধু debugging নয়, অনেক ক্ষেত্রে regulatory/আইনি প্রয়োজনও।

### ৬. `externalTrxId` field-টা রাখার উদ্দেশ্য কী?

**উত্তর:** `externalTrxId` হলো user যে bKash/Nagad/Bank লেনদেনে admin-কে টাকা পাঠিয়েছে, তার **বাইরের (external) TrxID**। Admin এই id দিয়ে নিজের bKash/Nagad statement-এ মিলিয়ে দেখে যে টাকাটা আসলেই এসেছে কিনা — এটাই approval-এর verification-এর মূল ভিত্তি। এটা না থাকলে admin কীভাবে নিশ্চিত হতো যে user সত্যিই টাকা পাঠিয়েছে? পাশাপাশি এটা একটা **reconciliation** tool — app-এর record আর payment provider-এর record মেলানোর সংযোগসূত্র। বাস্তবে এই field-এ duplicate check-ও থাকা উচিত, যাতে একই external TrxID দিয়ে দুটো request করে দুবার টাকা claim করা না যায়।

### ৭. `reject` তো আছেই — আলাদা `hold` status কেন দরকার?

**উত্তর:** `reject` আর `hold`-এর অর্থ ভিন্ন। `reject` (→ `FAILED`) মানে **চূড়ান্ত প্রত্যাখ্যান** — request বাতিল, শেষ। কিন্তু `hold` (→ `HOLD`) মানে **"এখন সিদ্ধান্ত নিচ্ছি না"** — হয়তো admin আরও যাচাই করতে চায়, user-এর সাথে কথা বলতে চায়, বা external payment এখনো নিশ্চিত হয়নি। Hold একটা মাঝামাঝি অবস্থা যেখানে request আটকে থাকে কিন্তু বাতিল হয় না, পরে আবার সিদ্ধান্ত নেওয়া যায়। বাস্তব finance workflow-এ এই "pending investigation" অবস্থাটা গুরুত্বপূর্ণ — সব কিছু হয় হ্যাঁ নয় না নয়, কিছু জিনিস আরও তথ্যের অপেক্ষায় থাকে।

### ৮. Approve-এর সময় `LedgerEntry`-টা main `$transaction`-এর পরে আলাদাভাবে তৈরি হয় কেন?

**উত্তর:** কারণ ledger entry-তে `transactionId` (foreign key) দরকার, কিন্তু সেই Transaction তৈরি হচ্ছে ঠিক ঐ একই transaction array-তে — আর array-form `$transaction`-এ একটা statement-এর ফলাফল (নতুন Transaction-এর id) সাথে সাথে পরের statement-এ ব্যবহার করা যায় না। তাই code প্রথমে Transaction তৈরি করে, তারপর তার `trxId` দিয়ে সেটা খুঁজে নিয়ে ledger entry বানায়। এটা এই code-এর একটা সীমাবদ্ধতা — আদর্শভাবে পুরোটা একটা interactive `$transaction(async (tx) => {...})` callback-এ রাখা উচিত (যেমন recharge/transfer-এ করা হয়েছে), যাতে ledger-ও একই atomic unit-এ থাকে এবং কোনো fাঁক না থাকে। এটা refactor করার একটা ভালো জায়গা।

### ৯. User `GET /api/add-money`-এ শুধু নিজের request পায় — এটা কীভাবে নিশ্চিত করা হয়?

**উত্তর:** Query-তে সবসময় session থেকে পাওয়া user-এর id দিয়ে filter করা হয়: `db.addMoneyRequest.findMany({ where: { userId: user.id }, ... })`। যেহেতু `user` আসে server-side session থেকে (client-controlled কিছু নয়), কেউ অন্য user-এর request দেখতে পারে না। এটা **broken object-level authorization (BOLA/IDOR)** নামের একটা সাধারণ দুর্বলতা ঠেকায় — যেখানে কেউ URL/parameter বদলে অন্যের data দেখে ফেলে। নিয়ম: list/detail endpoint-এ সবসময় "এই resource কি সত্যিই এই logged-in user-এর?" — সেটা server-side ownership filter দিয়ে নিশ্চিত করতে হবে, শুধু id দিয়ে খোঁজা যথেষ্ট নয়।

### ১০. এই add-money workflow-তে double-spend বা replay-এর ঝুঁকি কোথায় থাকতে পারে?

**উত্তর:** মূল ঝুঁকি approve ধাপে — যদি একই `PENDING` request প্রায় একই মুহূর্তে দুবার approve হয়ে যায় (দুই admin, বা double-click), তাহলে balance দুবার বাড়তে পারে। বর্তমান code `if (request.status !== "PENDING")` চেক দিয়ে এটা কমায়, কিন্তু এই চেক আর update আলাদা ধাপে হওয়ায় একটা সূক্ষ্ম race থেকে যায়। সঠিক সমাধান — status transition-টাকে atomic conditional update করা: `updateMany({ where: { id, status: "PENDING" }, data: { status: "SUCCESS" } })` এবং `count === 1` হলে তবেই balance বাড়ানো, সবই এক `$transaction`-এ। এছাড়া `externalTrxId`-এ unique constraint দিলে একই external payment দিয়ে দুটো request-ও ঠেকানো যায়। মূল নীতি — আর্থিক state transition সবসময় atomic ও idempotent হতে হবে।
