# ০২ — Recharge Feature (Transaction Pipeline)

## Feature-টা আসলে কী?

Agent তার wallet-এর balance থেকে যেকোনো mobile number-এ recharge করে। প্রতিটা সফল recharge-এ agent একটা **profit** (commission) পায়। এই feature-টা একটা **financial transaction pipeline** — এখানে টাকা কমছে, তাই এটা অবশ্যই নির্ভুল ও নিরাপদ হতে হবে (এক টাকাও এদিক-সেদিক হওয়া চলবে না)।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| UI | `src/app/(app)/recharge/page.tsx` | operator, number, amount নেয় |
| API | `src/app/api/recharge/route.ts` | পুরো recharge logic |
| Validation | `src/lib/validation/schemas.ts` → `rechargeSchema` | input যাচাই |
| DB | `Transaction`, `RechargeRequest`, `LedgerEntry`, `Wallet`, `AuditLog` | records |

### জড়িত file গুলো সংক্ষেপে

- **`src/app/(app)/recharge/page.tsx`** — recharge-এর UI: operator নির্বাচন, mobile number ও amount নেওয়ার form; submit করলে `/api/recharge`-এ POST পাঠায় ও ফলাফল (success/fail) দেখায়।
- **`src/app/api/recharge/route.ts`** — পুরো server-side logic: authentication, input validation, atomic balance debit, চারটা DB record তৈরি, mock provider call, আর fail হলে refund।
- **`src/lib/validation/schemas.ts → rechargeSchema`** — request body-র runtime validation: phone সঠিক format কিনা, amount একটা positive finite সংখ্যা কিনা, operator ঠিক আছে কিনা।
- **`Transaction` (DB)** — মূল লেনদেনের record; `trxId` unique (idempotency), status `PROCESSING → SUCCESS/FAILED`।
- **`RechargeRequest` (DB)** — provider-এর কাছে পাঠানোর জন্য আলাদা queue entry; retry/attempt tracking-এর জন্য।
- **`LedgerEntry` (DB)** — balance পরিবর্তনের হিসাব (DEBIT, আর fail হলে CREDIT refund)।
- **`Wallet` (DB)** — agent-এর balance; atomic decrement এখানেই হয়।
- **`AuditLog` (DB)** — কে কখন কত recharge করল তার log (`RECHARGE_INITIATE/SUCCESS/FAILED`)।

## পুরো flow-টা কীভাবে কাজ করে

### ধাপ ১ — Authentication ও Input Validation

প্রথমেই session থেকে user বের করা হয়। তারপর input **zod schema** দিয়ে যাচাই করা হয় — phone সঠিক format কিনা, amount একটা positive finite সংখ্যা কিনা।

```ts
const user = await getRequestUser(req);
if (!user) return unauthorized();

const parsed = rechargeSchema.safeParse(await req.json().catch(() => null));
if (!parsed.success) return badRequest(parsed.error.issues[0]?.message ?? "Invalid recharge data");
```

কেন zod দরকার? আগে code-টা `as { amount: number }` cast করত, কিন্তু JSON-এ কেউ `amount: "50"` বা `amount: Infinity` পাঠালে সেই cast কিছু ধরত না। zod runtime-এ আসল মান check করে, তাই type-confusion bug আটকায়।

### ধাপ ২ — Atomic, Overdraft-Safe Debit (সবচেয়ে গুরুত্বপূর্ণ অংশ)

টাকা কাটার আগে balance যথেষ্ট আছে কিনা দেখতে হয়। কিন্তু "আগে পড়ো, তারপর লেখো" pattern-এ একটা **race condition** থাকে — দুটো request একসাথে এলে দুটোই একই balance পড়ে দুটোই pass করে ফেলতে পারে (double-spend)। এটা এড়াতে balance check আর deduction একই সাথে, atomically করা হয়:

```ts
await db.$transaction(async (tx) => {
  const debit = await tx.wallet.updateMany({
    where: { userId: user.id, balance: { gte: amount } },  // যথেষ্ট থাকলেই
    data: { balance: { decrement: amount } },              // তবেই কাটো
  });
  if (debit.count === 0) throw new Error("INSUFFICIENT_BALANCE");
  // ... Transaction, RechargeRequest, LedgerEntry, AuditLog তৈরি
});
```

এখানে `where: { balance: { gte: amount } }` শর্তটা database-এর ভেতরে atomically চেক হয়, তাই দুটো concurrent request-এর একটাই সফল হবে — balance কখনো negative হবে না।

### ধাপ ৩ — একসাথে চারটা record তৈরি (`$transaction`)

একটা recharge মানে শুধু balance কমানো নয় — চারটা কাজ একসাথে হতে হবে, নাহলে data inconsistent হয়ে যাবে:

1. **Transaction** — মূল লেনদেনের record (status `PROCESSING`)
2. **RechargeRequest** — provider-এর কাছে পাঠানোর জন্য queue entry
3. **LedgerEntry** — hisab-nikash-এর জন্য DEBIT entry (balanceBefore/After সহ)
4. **AuditLog** — কে, কখন, কী করল তার log

`$transaction`-এর ভেতরে এই সবগুলো থাকায় — হয় সব হবে, নাহয় কিছুই হবে না (**all-or-nothing / atomicity**)।

### ধাপ ৪ — Provider Call ও Refund

এই demo-তে provider call mock করা (95% সফল)। যদি recharge **fail** করে, তাহলে কাটা টাকা atomically **refund** করা হয় এবং একটা CREDIT ledger entry বসানো হয়:

```ts
if (!success) {
  await db.$transaction(async (tx) => {
    const w = await tx.wallet.update({
      where: { userId: user.id },
      data: { balance: { increment: amount } },   // ফেরত
    });
    await tx.ledgerEntry.create({ /* CREDIT: RECHARGE_REFUND */ });
  });
}
```

### ধাপ ৫ — Profit Calculation ও Idempotency

প্রতি recharge-এ profit = amount-এর 3.5%। আর প্রতিটা transaction-এর একটা unique `trxId` থাকে (`makeTrxId("RCH")`) — এটা **idempotency key**, যাতে একই লেনদেন দুবার হিসাব না হয়।

## গুরুত্বপূর্ণ concept গুলো

- **Atomicity (ACID)**: একটা transaction হয় পুরোপুরি হবে, নাহয় একদমই হবে না।
- **Race condition**: একাধিক request একসাথে এলে যে bug হয় — DB-level conditional update দিয়ে ঠেকানো হয়।
- **Idempotency**: একই request দুবার এলে যেন দুবার প্রভাব না ফেলে।
- **Compensating transaction (refund)**: fail হলে আগের কাজ উল্টে দেওয়া।

---

## Interview Questions

### ১. `db.$transaction` কেন দরকার এখানে? না দিলে কী হতে পারত?

**উত্তর:** একটা recharge-এ চারটা আলাদা DB write হয় — balance কমানো, Transaction তৈরি, RechargeRequest তৈরি, LedgerEntry তৈরি। `$transaction` ছাড়া যদি balance কমানোর পর server crash করে বা DB error হয়, তাহলে টাকা তো কেটে গেল কিন্তু কোনো Transaction record রইল না — অর্থাৎ agent-এর টাকা হাওয়া, অথচ কোনো প্রমাণ নেই। `$transaction` এই সবগুলোকে **atomic** করে দেয়: হয় চারটাই commit হবে, নাহয় error হলে সব **rollback** হয়ে যাবে। ফলে database সবসময় একটা consistent অবস্থায় থাকে।

### ২. Race condition কী, আর `updateMany` দিয়ে overdraft কীভাবে ঠেকানো হয়েছে?

**উত্তর:** ধরো agent-এর balance ৳100, আর সে প্রায় একই মুহূর্তে দুটো ৳100-এর recharge দিল। পুরনো "আগে balance পড়ো (৳100), তারপর ৳100 কমাও" pattern-এ দুটো request-ই ৳100 পড়ে, দুটোই "যথেষ্ট আছে" ভেবে pass করে — ফলে ৳200 কাটা যায়, balance হয় ৳-100। এটাই race condition / double-spend। সমাধান: `updateMany({ where: { balance: { gte: amount } }, data: { balance: { decrement: amount } } })` — এখানে "যথেষ্ট আছে কিনা" শর্তটা database নিজে atomically একটা row lock করে চেক করে। তাই দুটো concurrent request-এর মধ্যে একটাই সফল হয় (`count === 1`), অন্যটার `count === 0` হয় এবং আমরা INSUFFICIENT_BALANCE throw করি। Balance কখনো negative হয় না।

### ৩. Recharge fail করলে refund কীভাবে handle করা হয়, আর এটাকে কী বলে?

**উত্তর:** Provider call fail করলে টাকা তো ইতিমধ্যে কেটে ফেলা হয়েছে, তাই আমরা একটা **compensating transaction** চালাই — atomically balance-এ টাকা `increment` করে ফেরত দিই এবং একটা `CREDIT` ledger entry (`RECHARGE_REFUND`) বসাই, যাতে হিসাবে দেখা যায় কেন balance আবার বাড়ল। এটাকে distributed systems-এ **Saga / compensating transaction** pattern বলে: মূল কাজ ব্যর্থ হলে তার প্রভাব উল্টে দেওয়া হয়, delete করে নয় বরং একটা নতুন উল্টো entry দিয়ে — যাতে পুরো audit trail অক্ষত থাকে।

### ৪. `trxId` (idempotency key)-এর ভূমিকা কী?

**উত্তর:** `trxId` হলো প্রতিটা লেনদেনের unique identifier, আর schema-তে এটা `@unique`। এর কাজ হলো **idempotency** নিশ্চিত করা — অর্থাৎ network glitch-এ যদি একই recharge request দুবার চলে আসে, তবু যেন দুবার টাকা না কাটে বা দুবার হিসাব না হয়। যেহেতু `trxId` unique, একই id দিয়ে দ্বিতীয় লেনদেন তৈরি করতে গেলে DB constraint আটকে দেবে। Real payment system-এ এই idempotency key অত্যন্ত জরুরি, কারণ retry/timeout খুবই সাধারণ ঘটনা।

### ৫. Input validation-এ শুধু TypeScript type (`as`) যথেষ্ট নয় কেন, zod কী বাড়তি দেয়?

**উত্তর:** TypeScript-এর type শুধু **compile-time**-এ কাজ করে; runtime-এ `as { amount: number }` কেবল compiler-কে "বিশ্বাস করো এটা number" বলে, কিন্তু আসলে যাচাই করে না। ফলে API-তে কেউ JSON-এ `amount: "50"`, `amount: -100`, বা `amount: Infinity` পাঠালে সেটা type cast পেরিয়ে যায় এবং হিসাব ভেঙে দিতে পারে (যেমন `amount < 10` চেক string-এ ভুল behave করে)। **zod** runtime-এ আসল মান validate করে — number কিনা, finite কিনা, positive কিনা, দশমিকের ঘর ঠিক আছে কিনা — সব চেক করে, নাহলে সাথে সাথে 400 error দেয়। তাই money-handling endpoint-এ zod-এর মতো runtime validation বাধ্যতামূলক।

### ৬. Transaction-এই তো সব তথ্য আছে — আলাদা `RechargeRequest` model কেন?

**উত্তর:** দুটোর দায়িত্ব আলাদা। `Transaction` হলো **আর্থিক record** — টাকা কাটা হয়েছে, ledger-এর সাথে যুক্ত, এটা মূলত accounting-এর জন্য। আর `RechargeRequest` হলো **operational/queue record** — provider-এর কাছে আসলে recharge পাঠানোর কাজটা track করে: কতবার চেষ্টা হলো (`attempts`), পরের retry কখন (`nextRetryAt`), provider কী error দিল (`errorMessage`), webhook payload কী ছিল। বাস্তবে recharge একটা background worker provider API-তে পাঠায় এবং fail হলে retry করে — এই সব state আর্থিক Transaction-এ মেশালে model জটিল ও নোংরা হতো। **Separation of concerns**: টাকার হিসাব এক জায়গায়, delivery/retry-র অবস্থা আরেক জায়গায়।

### ৭. `profit` কেন server-এ হিসাব করা হয়, client পাঠালে সমস্যা কী?

**উত্তর:** Profit (commission) হিসাব হয় server-এ — `profit = amount * 0.035`। যদি এটা client পাঠাত, তাহলে একজন agent request body-তে `profit: 999999` বসিয়ে নিজের আয় ইচ্ছেমতো বাড়িয়ে দিতে পারত — সরাসরি আর্থিক জালিয়াতি। সাধারণ নীতি: **টাকা-সম্পর্কিত কোনো হিসাব কখনো client-এর ওপর ছাড়া যাবে না**; rate, fee, commission, total — সবই server-এ, বিশ্বস্ত উৎস থেকে হিসাব হতে হবে। Client শুধু কী করতে চায় তা জানায় (কত টাকা, কোন number), আর "এতে কত profit/fee" — সেই সিদ্ধান্ত সবসময় server নেয়।

### ৮. Provider call (mock)-টা `$transaction`-এর বাইরে কেন, ভেতরে রাখলে সমস্যা কী?

**উত্তর:** Provider call একটা **external network operation** যা কয়েক সেকেন্ড লাগতে পারে বা টাইম-আউট হতে পারে। এটা DB `$transaction`-এর ভেতরে রাখলে পুরো transaction ঐ সময় ধরে DB row lock করে বসে থাকত, যা concurrency মারাত্মকভাবে কমিয়ে দিত এবং deadlock-এর ঝুঁকি বাড়াত। তাই design-টা এমন: প্রথমে দ্রুত atomic transaction-এ balance কাটা ও record তৈরি হয় (status `PROCESSING`), তারপর transaction commit হওয়ার পর ধীরগতির provider call হয়, শেষে result অনুযায়ী status আলাদাভাবে `SUCCESS/FAILED`-এ update হয়। এটাই সঠিক pattern — **দ্রুত/atomic কাজ transaction-এ, ধীর/external কাজ তার বাইরে**।

### ৯. Recharge fail হলে refund হয় — কিন্তু একই fail-এ দুবার refund (double refund) কীভাবে ঠেকানো যায়?

**উত্তর:** বর্তমান code-এ প্রতিটা recharge request-এর জীবনচক্র একবারই চলে, তাই সরল ক্ষেত্রে double refund হয় না। কিন্তু production-এ যেখানে retry/webhook থাকে, ঝুঁকি বাড়ে — একই fail event দুবার এলে দুবার refund হয়ে যেতে পারে। এটা ঠেকানোর উপায়: (ক) refund দেওয়ার আগে status atomically চেক করা — যেমন `updateMany({ where: { id, status: "FAILED", refundedAt: null }, ... })`, যাতে একবার refund হলে দ্বিতীয়বার শর্ত না মেলে; (খ) refund ledger entry-তে একটা unique reference (`RECHARGE_REFUND:trxId`) রাখা এবং সেটা `@unique` করা, যাতে একই refund দুবার লেখা DB constraint আটকে দেয়। মূলত refund-কেও idempotent করতে হবে, ঠিক মূল লেনদেনের মতো।

### ১০. Invalid বা অজানা operator এলে কী হয়, আর operator কীভাবে normalize করা হয়?

**উত্তর:** Client operator string পাঠায় (যেমন "gp", "robi"), server সেটাকে uppercase করে (`operator?.toUpperCase()`) এবং schema-র enum-এর সাথে মেলায়; না মিললে বা না দিলে সেটা `"UNKNOWN"` হিসেবে ধরা হয়। ফলে অজানা operator-এও system crash করে না, লেনদেন `UNKNOWN` operator দিয়ে রেকর্ড হয় (পরে detect/সংশোধন করা যায়)। এটা একটা **defensive/graceful-degradation** approach — অপ্রত্যাশিত input-এ error না ছুঁড়ে একটা নিরাপদ default-এ পড়ে যাওয়া। তবে কড়া করতে চাইলে zod schema-তে operator-কে একটা নির্দিষ্ট enum-এ সীমাবদ্ধ করে অবৈধ মান সরাসরি 400 দিয়ে ফেরানো যেত।
