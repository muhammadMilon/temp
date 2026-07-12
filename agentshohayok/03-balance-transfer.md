# ০৩ — Balance Transfer (Agent থেকে Agent)

## Feature-টা আসলে কী?

একজন agent তার wallet থেকে আরেকজন agent/user-এর কাছে balance পাঠাতে পারে (bKash-এর "Send Money"-র মতো)। এখানে একই সাথে **দুটো wallet** বদলায় — sender-এর balance কমে, receiver-এর বাড়ে। তাই এটা double-entry accounting-এর একটা চমৎকার উদাহরণ।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| UI | `src/app/(app)/transfer/page.tsx` | receiver phone, amount নেয় |
| API | `src/app/api/transfer/route.ts` | পুরো transfer logic |
| Validation | `src/lib/validation/schemas.ts` → `transferSchema` | input যাচাই |
| DB | `Transfer`, `Transaction`, `LedgerEntry`, `Wallet`, `AuditLog` | records |

### জড়িত file গুলো সংক্ষেপে

- **`src/app/(app)/transfer/page.tsx`** — transfer-এর UI: receiver-এর phone ও amount নেওয়ার form, সাথে recent transfer history দেখানো।
- **`src/app/api/transfer/route.ts`** — server-side logic: validation, নিজের কাছে transfer আটকানো, এক `$transaction`-এ sender debit + receiver credit + ledger + audit।
- **`src/lib/validation/schemas.ts → transferSchema`** — receiverPhone, amount, notes-এর runtime validation।
- **`Transfer` (DB)** — transfer-নির্দিষ্ট record (sender, receiverPhone, receiverId, status)।
- **`Transaction` (DB)** — একই ঘটনার সাধারণ আর্থিক record, history/report-এ দেখানোর জন্য।
- **`LedgerEntry` (DB)** — sender-এর DEBIT ও receiver-এর CREDIT — double-entry।
- **`Wallet` (DB)** — দুই পক্ষের balance এখানে বদলায় (atomic decrement/increment)।
- **`AuditLog` (DB)** — `TRANSFER_SUCCESS` action-এর log।

## পুরো flow-টা কীভাবে কাজ করে

### ধাপ ১ — Validation ও নিজের কাছে transfer আটকানো

```ts
const parsed = transferSchema.safeParse(await req.json().catch(() => null));
if (!parsed.success) return badRequest(...);

const receiverPhone = normalizePhone(parsed.data.receiverPhone);
if (receiverPhone === user.phone) return badRequest("Cannot transfer to yourself");
```

Phone number নানা format-এ আসতে পারে (`+8801...`, `01...`, `8801...`), তাই `normalizePhone` দিয়ে সবাইকে একটা standard format-এ আনা হয় — নাহলে "নিজের কাছে transfer" check বা receiver খোঁজা ভুল হতে পারে।

### ধাপ ২ — এক transaction-এর ভেতরে debit + credit

সবচেয়ে গুরুত্বপূর্ণ নিয়ম: **sender-এর টাকা কাটা আর receiver-কে টাকা দেওয়া — দুটো একই `$transaction`-এর ভেতরে হতে হবে।** নাহলে sender-এর কাটা গেল কিন্তু receiver পেল না — টাকা হাওয়া।

```ts
const balanceAfter = await db.$transaction(async (tx) => {
  // Sender debit — atomic ও overdraft-safe
  const debit = await tx.wallet.updateMany({
    where: { userId: user.id, balance: { gte: amount } },
    data: { balance: { decrement: amount } },
  });
  if (debit.count === 0) throw new Error("INSUFFICIENT_BALANCE");

  // Sender-এর নতুন balance পড়ে ledger বানানো
  const senderWallet = await tx.wallet.findUniqueOrThrow({ where: { userId: user.id } });
  const senderAfter  = Number(senderWallet.balance);
  const senderBefore = senderAfter + amount;

  // Transfer, Transaction, DEBIT ledger, AuditLog তৈরি ...

  // Receiver থাকলে তার wallet-এ credit
  if (receiver) {
    const receiverWallet = await tx.wallet.upsert({
      where: { userId: receiver.id },
      create: { userId: receiver.id, balance: amount },
      update: { balance: { increment: amount } },
    });
    // CREDIT ledger entry (TRANSFER_IN) ...
  }
  return senderAfter;
});
```

### ধাপ ৩ — Receiver না থাকলে কী হয়?

Receiver যদি system-এ registered না থাকে (`receiver == null`), তখন transfer record হয় (`receiverId: null`), sender-এর টাকা কাটে, কিন্তু কারো wallet-এ credit হয় না — টাকাটা "pending/unclaimed" হিসেবে ধরা হয়। বাস্তব app-এ এখানে receiver-কে notify করে later claim করানো হতো।

### ধাপ ৪ — Error handling

`INSUFFICIENT_BALANCE` throw হলে পুরো transaction rollback হয়ে যায় (কিছুই বদলায় না) এবং user একটা পরিষ্কার 400 error পায়:

```ts
catch (err) {
  if (err instanceof Error && err.message === "INSUFFICIENT_BALANCE")
    return badRequest("Insufficient balance");
  throw err;
}
```

## গুরুত্বপূর্ণ concept গুলো

- **Double-entry**: প্রতিটা টাকা চলাচলে একদিকে DEBIT, আরেকদিকে CREDIT — যোগফল সবসময় শূন্য (টাকা তৈরি/ধ্বংস হয় না, শুধু হাত বদলায়)।
- **Atomic multi-record update**: একাধিক wallet একসাথে বদলাতে হলে এক transaction-এ রাখতে হয়।
- **Data normalization**: input (phone) একটা standard রূপে আনা।

---

## Interview Questions

### ১. Sender-এর debit আর receiver-এর credit কেন একই transaction-এ রাখা জরুরি?

**উত্তর:** কারণ টাকা "সংরক্ষিত" থাকতে হবে — sender থেকে যত কমবে, receiver-এ ঠিক ততই বাড়বে। যদি এই দুটো আলাদা আলাদা operation হতো এবং sender-এর debit হওয়ার পর (কিন্তু receiver-এর credit হওয়ার আগে) server crash করত, তাহলে টাকাটা কারো কাছেই থাকত না — system থেকে টাকা উবে যেত। একই `$transaction`-এ রাখলে হয় দুটোই ঘটবে, নাহয় কোনোটাই ঘটবে না (atomicity), তাই টাকার মোট পরিমাণ সবসময় ঠিক থাকে।

### ২. `normalizePhone` কেন দরকার, এটা না থাকলে কোন কোন bug হতে পারত?

**উত্তর:** একই নম্বর user নানাভাবে লিখতে পারে — `01711...`, `+88 01711...`, `8801711...`। এগুলোকে standard রূপে না আনলে দুটো সমস্যা: (ক) "নিজের কাছে transfer করা যাবে না" check ফাঁকি দেওয়া যেত — কেউ নিজের নম্বর অন্য format-এ দিয়ে check এড়িয়ে যেতে পারত; (খ) receiver খোঁজার সময় DB-তে normalized রূপে সংরক্ষিত নম্বরের সাথে না মিললে registered receiver-কেও "not found" ধরা হতো, ফলে তার wallet-এ credit হতো না। `normalizePhone` সবাইকে এক রূপে এনে এই দুই সমস্যা দূর করে।

### ৩. Receiver system-এ না থাকলে transfer কীভাবে handle হয়, আর এটা নিরাপদ কেন?

**উত্তর:** Receiver `null` হলে `Transfer` record তৈরি হয় `receiverId: null` দিয়ে, sender-এর balance কাটে, কিন্তু কোনো wallet-এ credit হয় না — টাকাটা কার্যত "unclaimed" অবস্থায় থাকে (record থেকে পরে claim করানো যায়)। এটা নিরাপদ কারণ পুরো ব্যাপারটা এখনো একই `$transaction`-এর ভেতরে — sender-এর টাকা কাটা আর Transfer record তৈরি atomic। ফলে কোনো টাকা "হারিয়ে" যায় না; audit trail-এ স্পষ্ট থাকে যে টাকা পাঠানো হয়েছে কিন্তু কেউ এখনো পায়নি।

### ৪. এখানে "double-entry accounting" বলতে কী বোঝায়?

**উত্তর:** Double-entry মানে প্রতিটা টাকা চলাচলকে দুইটা মিলিত entry দিয়ে লেখা — এক wallet থেকে DEBIT (কমা), আরেক wallet-এ CREDIT (বাড়া)। এই project-এ transfer-এ sender-এর জন্য `TRANSFER` DEBIT ledger এবং receiver-এর জন্য `TRANSFER_IN` CREDIT ledger তৈরি হয়, দুটোই একই `trxId`-তে যুক্ত। সুবিধা হলো — যেকোনো সময় সব ledger যোগ করলে হিসাব মিলবে, এবং কোনো balance কেন পরিবর্তন হলো তার সম্পূর্ণ ইতিহাস থাকে। এটাই ব্যাংকিং/fintech-এ হিসাব নির্ভুল রাখার প্রমাণিত পদ্ধতি।

### ৫. `updateMany` দিয়ে atomic debit না করে যদি `update` দিয়ে সরাসরি নতুন balance বসানো হতো, সমস্যা কী?

**উত্তর:** Prisma-র `update` একটা নির্দিষ্ট row খুঁজে সেটার মান বসায়, কিন্তু "balance যথেষ্ট থাকলে তবেই কমাও" — এই শর্ত `update`-এর `where`-এ (unique field ছাড়া) দেওয়া যায় না। তাই আগে balance পড়ে, JavaScript-এ হিসাব করে, তারপর নতুন absolute মান বসাতে হতো — আর এই "পড়ো তারপর লেখো" fাঁক দিয়েই race condition ঢোকে (দুটো concurrent transfer একই balance পড়ে দুটোই সফল হয়ে overdraft করে ফেলে)। `updateMany({ where: { balance: { gte: amount } }, data: { decrement } })` শর্ত ও পরিবর্তনকে database-এর ভেতরে atomically একত্রে করে, তাই concurrency-তেও balance কখনো negative হয় না।

### ৬. একই transfer-এর জন্য `Transfer` আর `Transaction` — দুটো record কেন তৈরি হয়?

**উত্তর:** দুটোর দৃষ্টিভঙ্গি আলাদা। `Transfer` হলো একটা **domain-specific** record — শুধু transfer-সম্পর্কিত তথ্য রাখে (কে পাঠাল, কার নম্বরে, receiver registered কিনা)। আর `Transaction` হলো একটা **সাধারণ, একীভূত (unified)** আর্থিক record — recharge, transfer, add-money — সব ধরনের লেনদেন এই এক model-এ আসে, তাই user-এর "সব লেনদেনের history" এক জায়গা থেকে সহজে দেখানো যায়। দুটোই একই `trxId` ভাগ করে, তাই প্রয়োজনে মেলানো যায়। এটা একটা সচেতন design choice — বিশেষায়িত view (Transfer) আর সাধারণ reporting view (Transaction) — দুটোই পাওয়া যায়।

### ৭. Receiver-কে `upsert` দিয়ে credit করা হয়, শুধু `update` নয় কেন?

**উত্তর:** কারণ receiver-এর হয়তো এখনো কোনো `Wallet` row নাও থাকতে পারে (নতুন user বা যার wallet কখনো তৈরি হয়নি)। `update` করলে row না থাকলে error হতো। `upsert` দুটো কাজ একসাথে করে — wallet থাকলে balance `increment` করে, না থাকলে ঐ amount দিয়ে নতুন wallet **create** করে। এতে "receiver-এর wallet আছে কিনা" আগে থেকে চেক করার আলাদা code লাগে না, এবং race condition-ও এড়ানো যায় (একই সময়ে দুটো transfer এলে দুটোই নিরাপদে handle হয়)। মূলত upsert হলো "create or update" — যেখানে existence অনিশ্চিত সেখানে এটা নিরাপদ ও পরিচ্ছন্ন।

### ৮. Admin overview-তে ৳50,000-এর বেশি transfer "suspicious" ধরা হয় — এই fraud detection কীভাবে আরও ভালো করা যায়?

**উত্তর:** বর্তমানে একটা সরল threshold (`amount >= 50000`) দিয়ে বড় transfer-কে suspicious চিহ্নিত করা হয়। এটা প্রাথমিক পদক্ষেপ, কিন্তু সহজে ফাঁকি দেওয়া যায় (কেউ ৳49,999 করে অনেকবার পাঠাতে পারে — "structuring")। উন্নত করার উপায়: (ক) **velocity check** — অল্প সময়ে অনেকগুলো transfer বা একই receiver-এ বারবার; (খ) **pattern/anomaly detection** — user-এর স্বাভাবিক আচরণের তুলনায় হঠাৎ বড় পরিবর্তন; (গ) **new-receiver risk** — একদম নতুন/unverified নম্বরে বড় অঙ্ক; (ঘ) rule-based scoring বা ML model দিয়ে risk score বের করে থ্রেশহোল্ড পেরোলে hold/manual review। মূল কথা — শুধু single amount নয়, সময়, ফ্রিকোয়েন্সি ও প্রেক্ষাপট মিলিয়ে বিচার করা।

### ৯. একই transfer পরপর দুবার submit হয়ে গেলে (double submit) কী হয়, এটা কীভাবে সামলানো উচিত?

**উত্তর:** এই code-এ প্রতিবার নতুন `trxId` তৈরি হয় (`makeTrxId("TRF")`), তাই দুবার submit করলে দুটো আলাদা transfer হবে এবং দুবার টাকা কাটবে — যা ব্যবহারকারীর জন্য অনাকাঙ্ক্ষিত। বাস্তব system-এ এটা ঠেকাতে **client-provided idempotency key** ব্যবহার করা হয়: client প্রতিটা "logical" transfer-এর জন্য একটা unique key পাঠায়, server সেই key `@unique` হিসেবে রাখে; একই key-তে দ্বিতীয় request এলে নতুন transfer না করে আগের result ফেরত দেয়। পাশাপাশি frontend-এ submit বাটন disable করা ও double-click ঠেকানো সহায়ক প্রথম স্তর। সংক্ষেপে — money action-এ সবসময় idempotency key দিয়ে "exactly once" নিশ্চিত করা উচিত।

### ১০. Sender-এর debit হয়ে গেল কিন্তু receiver-এর credit করতে গিয়ে error হলো — তখন কী হয়?

**উত্তর:** কিছুই "আধা-সম্পন্ন" থাকবে না, কারণ পুরো debit + credit একই `db.$transaction`-এর ভেতরে। receiver-এর credit-এ যদি কোনো error/exception হয়, তাহলে transaction commit-ই হবে না — Prisma পুরোটা **rollback** করে দেবে, ফলে sender-এর কাটা টাকাও ফিরে আসবে। অর্থাৎ হয় দুই দিকই ঘটবে, নাহয় কোনোটাই ঘটবে না (atomicity)। এই কারণেই cross-wallet operation একই transaction-এ রাখা অপরিহার্য — না রাখলে sender-এর টাকা কেটে যেত অথচ receiver পেত না, আর system থেকে টাকা হারিয়ে যেত।
