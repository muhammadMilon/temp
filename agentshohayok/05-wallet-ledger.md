# ০৫ — Wallet ও Ledger System (Double-Entry Accounting)

## Feature-টা আসলে কী?

প্রতিটা agent-এর একটা **Wallet** আছে যেখানে তার current balance থাকে। কিন্তু শুধু একটা balance number রাখা যথেষ্ট নয় — balance কেন, কখন, কীভাবে বদলাল তার প্রতিটা ধাপের হিসাব রাখতে হয়। এই হিসাবের খাতাই **Ledger** (`LedgerEntry`)।

মূল নীতি (schema-তেই লেখা আছে): **প্রতিটা balance পরিবর্তনে অবশ্যই একটা LedgerEntry তৈরি হবে — কখনো সরাসরি balance বদলানো যাবে না হিসাব ছাড়া।**

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| DB Schema | `prisma/schema.prisma` → `Wallet`, `LedgerEntry` | মূল model |
| ব্যবহার | `api/recharge`, `api/transfer`, `api/add-money`, `api/admin/add-money/[id]` | প্রতিটা টাকা চলাচলে ledger লেখে |
| Helper | `src/lib/api-helpers.ts` → `ensureWallet` | wallet না থাকলে বানায় |

### জড়িত file গুলো সংক্ষেপে

- **`prisma/schema.prisma → Wallet`** — প্রতি user-এর balance, openingBalance, frozenAmount, currency, monthlyGoal রাখে; এটাই "এখন কত টাকা" জানার জায়গা।
- **`prisma/schema.prisma → LedgerEntry`** — প্রতিটা balance পরিবর্তনের একটা করে অপরিবর্তনীয় (append-only) হিসাব-লাইন; type (DEBIT/CREDIT), amount, balanceBefore/After, reference।
- **`api/recharge`, `api/transfer`, `api/add-money`, `api/admin/add-money/[id]`** — এই route-গুলো যখনই balance বদলায়, একই `$transaction`-এ একটা করে ledger entry লেখে (কখনো ledger ছাড়া balance বদলায় না)।
- **`src/lib/api-helpers.ts → ensureWallet`** — কোনো user-এর wallet না থাকলে 0 balance দিয়ে তৈরি করে দেয়, যাতে পরের operation নিরাপদে চলে।

## Data Model বোঝা

### Wallet
```prisma
model Wallet {
  balance         Decimal   @default(0) @db.Decimal(15, 2)
  openingBalance  Decimal   @default(0) @db.Decimal(15, 2)
  frozenAmount    Decimal   @default(0) @db.Decimal(15, 2)
  currency        String    @default("BDT")
  ledgerEntries   LedgerEntry[]
}
```

### LedgerEntry (হিসাবের প্রতিটা লাইন)
```prisma
model LedgerEntry {
  type           LedgerType   // DEBIT বা CREDIT
  amount         Decimal      @db.Decimal(15, 2)
  balanceBefore  Decimal      @db.Decimal(15, 2)   // পরিবর্তনের আগে
  balanceAfter   Decimal      @db.Decimal(15, 2)   // পরিবর্তনের পরে
  reference      String       // "RECHARGE:trx_abc123"
  transactionId  String?
}
```

## এটা কীভাবে কাজ করে

### প্রতিটা balance পরিবর্তন = একটা ledger line

যখনই recharge/transfer/add-money-তে balance বদলায়, একই `$transaction`-এর ভেতরে একটা `LedgerEntry` তৈরি হয় যেখানে থাকে — কত টাকা (`amount`), আগে কত ছিল (`balanceBefore`), পরে কত হলো (`balanceAfter`), আর কেন (`reference`)।

```ts
await tx.ledgerEntry.create({
  data: {
    walletId: wallet.id,
    type: "DEBIT",
    amount,
    balanceBefore,
    balanceAfter,
    reference: `RECHARGE:${trxId}`,
    description: `Recharge ${phone} — ${op}`,
    transactionId: txRec.id,
  },
});
```

### কেন `balanceBefore` ও `balanceAfter` দুটোই রাখা হয়?

এতে প্রতিটা entry নিজে থেকেই সম্পূর্ণ (self-contained) — শুধু একটা line দেখেই বোঝা যায় ঐ মুহূর্তে balance কত ছিল ও কত হলো। কোনো হিসাবে গোলমাল হলে ledger ধরে ধরে audit করা যায়, এমনকি balance আবার নতুন করে হিসাব (reconstruct) করা যায়।

### কেন `Decimal`, `Float` নয়?

টাকার হিসাবে `Decimal(15, 2)` ব্যবহার করা হয়েছে, `Float` নয়। কারণ floating-point-এ `0.1 + 0.2 === 0.30000000000000004` — অর্থাৎ টাকা নিয়ে rounding error হয়, যা fintech-এ কখনো গ্রহণযোগ্য নয়। `Decimal` সঠিক দশমিক হিসাব রাখে।

## গুরুত্বপূর্ণ concept গুলো

- **Immutable ledger**: ledger entry কখনো edit/delete হয় না, শুধু নতুন entry যোগ হয় (append-only) — তাই ইতিহাস অক্ষত।
- **Source of truth**: বিতর্ক হলে ledger-ই চূড়ান্ত সত্য, `balance` field শুধু quick-read-এর জন্য cache-এর মতো।
- **Decimal money**: টাকা সবসময় fixed-precision decimal-এ রাখা।

---

## Interview Questions

### ১. শুধু `Wallet.balance` একটা number রাখলেই তো চলত — আলাদা Ledger কেন দরকার?

**উত্তর:** শুধু balance রাখলে তুমি জানবে *এখন কত টাকা আছে*, কিন্তু জানবে না *কীভাবে এই অঙ্কে পৌঁছালাম*। কোনো ভুল হলে (যেমন balance অপ্রত্যাশিতভাবে কমে গেল) তদন্তের কোনো উপায় থাকবে না। Ledger একটা **append-only history** — প্রতিটা পরিবর্তনের আলাদা রেকর্ড। এতে (ক) audit করা যায়, (খ) বিতর্ক মেটানো যায় ("এই ৳500 কেন কাটল?" → ledger দেখাও), (গ) balance corrupt হলে ledger যোগ করে আবার হিসাব করা যায়। ব্যাংক কখনো শুধু balance রাখে না, প্রতিটা transaction-এর ledger রাখে — এই project সেই নীতি অনুসরণ করে।

### ২. টাকার জন্য `Decimal` ব্যবহার করা হয়েছে কেন, `Float`/`Double` নয়?

**উত্তর:** Floating-point সংখ্যা binary-তে সব দশমিক নিখুঁতভাবে রাখতে পারে না — যেমন `0.1 + 0.2` হয় `0.30000000000000004`। হাজার হাজার লেনদেনে এই ছোট ছোট rounding error জমে গিয়ে হিসাবে বড় গরমিল তৈরি করে, যা আর্থিক system-এ সম্পূর্ণ অগ্রহণযোগ্য। `Decimal(15, 2)` fixed precision-এ (২ দশমিক ঘর) সঠিক মান রাখে, তাই প্রতিটা টাকা-পয়সা নিখুঁত থাকে। সাধারণ নিয়ম: **money = decimal/integer (পয়সা এককে), কখনো float নয়।**

### ৩. `balanceBefore` আর `balanceAfter` — দুটোই কেন প্রতিটা entry-তে রাখা হয়?

**উত্তর:** এটা প্রতিটা ledger line-কে **self-contained** করে তোলে — একটা মাত্র entry দেখেই বোঝা যায় লেনদেনের আগে balance কত ছিল আর পরে কত হলো, অন্য কোনো row-এর ওপর নির্ভর করতে হয় না। এর ফায়দা: (ক) audit সহজ হয়; (খ) ledger-এ কোনো entry বাদ পড়েছে বা ভুল হয়েছে কিনা তা `আগেরটার balanceAfter == পরেরটার balanceBefore` মিলিয়ে ধরা যায় (consistency check); (গ) কোনো রিপোর্টে দ্রুত দেখানো যায়। এটা মূলত নির্ভুলতা যাচাইয়ের একটা built-in safeguard।

### ৪. "প্রতিটা balance পরিবর্তনে ledger লিখতেই হবে, সরাসরি balance update নয়" — এই নিয়মের কারণ কী?

**উত্তর:** যদি code-এর কোনো জায়গায় ledger ছাড়া সরাসরি `wallet.balance` বদলানো যায়, তাহলে সেই পরিবর্তনের কোনো হিসাব থাকবে না — audit trail-এ ফাঁক তৈরি হবে, আর ঠিক সেই ফাঁকেই bug বা fraud লুকিয়ে যেতে পারে। নিয়ম করে দিলে (balance পরিবর্তন = অবশ্যই ledger entry, একই transaction-এ) প্রতিটা পরিবর্তন explainable ও traceable থাকে। এটা একটা **invariant** — system-এর এমন একটা সত্য যা সবসময় বজায় থাকতেই হবে, নাহলে data-র বিশ্বাসযোগ্যতা নষ্ট।

### ৫. Ledger-কে "immutable / append-only" রাখা মানে কী, আর ভুল হলে ঠিক করা হয় কীভাবে?

**উত্তর:** Immutable/append-only মানে কোনো ledger entry একবার লেখা হলে সেটা আর edit বা delete করা হয় না — শুধু নতুন entry যোগ করা হয়। কোনো লেনদেন ভুল হলে সেই entry মুছে না ফেলে, একটা **উল্টো (reversing) entry** যোগ করা হয় (যেমন recharge fail হলে `RECHARGE_REFUND` CREDIT entry)। এতে পুরো ইতিহাস অক্ষত থাকে — "ভুল হয়েছিল এবং তা সংশোধন করা হয়েছে" — দুটোই রেকর্ডে দেখা যায়। এটাই সঠিক accounting practice: history পাল্টানো নয়, বরং সংশোধনকেও ইতিহাসের অংশ করা।

### ৬. `openingBalance` আর `frozenAmount` — এই আলাদা field দুটো কেন?

**উত্তর:** এগুলো `balance`-এর চেয়ে আলাদা তথ্য রাখে। `openingBalance` হলো agent শুরুতে যে balance নিয়ে যাত্রা করেছিল — এটা পরে profit/growth হিসাব করতে বা রিপোর্টিংয়ে "কত থেকে শুরু, এখন কত" দেখাতে কাজে লাগে। `frozenAmount` হলো balance-এর যে অংশটা সাময়িকভাবে **আটকে/সংরক্ষিত** রাখা হয়েছে — যেমন একটা pending transfer বা বিতর্কিত লেনদেনের জন্য টাকা "hold" করা, যাতে সেটা অন্য কাজে খরচ না হয়। বাস্তব wallet-এ available balance = total balance − frozen। এই আলাদা field-গুলো একটা wallet-কে নিছক একটা number-এর বদলে একটা পূর্ণাঙ্গ আর্থিক অবস্থা হিসেবে উপস্থাপন করে।

### ৭. Ledger entry-র `reference` field (যেমন `"RECHARGE:trx_abc123"`) কী কাজে লাগে?

**উত্তর:** `reference` প্রতিটা ledger line-কে তার **উৎস ঘটনার সাথে যুক্ত** করে — কেন এই DEBIT/CREDIT হলো তা এক নজরে বোঝা যায় (recharge, transfer, refund, add-money ইত্যাদি)। এটা একটা human-readable ও searchable সূত্র: কোনো balance পরিবর্তন নিয়ে প্রশ্ন উঠলে reference দেখে সরাসরি মূল লেনদেনে পৌঁছানো যায়। এছাড়া reporting/audit-এ ধরন অনুযায়ী গোছানো (grouping) সহজ হয় — যেমন "সব refund" বা "সব transfer" ledger খুঁজে বের করা। সংক্ষেপে, reference হলো হিসাব আর ঘটনার মধ্যে সেতুবন্ধন।

### ৮. Ledger থেকে balance আবার হিসাব (reconstruct) করা মানে কী, আর কখন এটা দরকার হয়?

**উত্তর:** যেহেতু প্রতিটা ledger entry-তে amount ও type (DEBIT/CREDIT) থাকে, তাই openingBalance থেকে শুরু করে সব entry ক্রমানুসারে যোগ/বিয়োগ করলে যেকোনো মুহূর্তের balance বের করা যায় — এটাই reconstruction। এটা দরকার হয় যখন (ক) `Wallet.balance` (যা মূলত quick-read cache) কোনো bug-এ ভুল হয়ে যায় এবং সঠিক মান আবার নির্ণয় করতে হয়; (খ) audit/verification-এ "balance আর ledger মেলে কিনা" পরীক্ষা করতে হয়; (গ) কোনো নির্দিষ্ট তারিখে balance কত ছিল তা জানতে হয়। এই কারণেই ledger-কে **source of truth** ধরা হয় আর balance field-কে তার একটা দ্রুত-পঠনযোগ্য সংক্ষিপ্তরূপ।

### ৯. `Wallet`-এ `currency` field রাখা হয়েছে কেন, এখন তো সব BDT?

**উত্তর:** এটা **ভবিষ্যৎ-প্রস্তুতি (future-proofing)**। এখন সব wallet BDT-তে, কিন্তু currency field রাখায় পরে multi-currency support যোগ করা অনেক সহজ হবে — schema বড় করে migrate না করেই। পাশাপাশি এটা একটা ভালো অভ্যাস: টাকার অঙ্কের সাথে সবসময় তার currency স্পষ্টভাবে যুক্ত রাখা, যাতে ভবিষ্যতে বা কোনো integration-এ "এই 100 কীসের 100?" — এই দ্ব্যর্থতা না হয়। তবে সত্যিকারের multi-currency করতে হলে currency-ভিত্তিক আলাদা balance, exchange rate, আর cross-currency ledger handling-ও লাগবে — currency field সেই ভিতটা তৈরি রাখে।

### ১০. অনেক concurrent লেনদেনের মধ্যেও wallet balance-এর নির্ভুলতা কীভাবে নিশ্চিত থাকে?

**উত্তর:** দুইটা স্তরে। প্রথমত, প্রতিটা balance পরিবর্তন একটা DB `$transaction`-এর ভেতরে হয়, তাই একটা লেনদেনের মাঝপথে অন্য লেনদেন হস্তক্ষেপ করে inconsistent অবস্থা তৈরি করতে পারে না (isolation)। দ্বিতীয়ত, debit-এ atomic conditional update ব্যবহার হয় — `updateMany({ where: { balance: { gte: amount } }, data: { decrement } })` — যেখানে "যথেষ্ট আছে কিনা" যাচাই আর কমানো একই atomic operation-এ database-এর row lock-এর অধীনে হয়। ফলে দুটো concurrent debit একই balance নিয়ে দুটোই সফল হতে পারে না (race condition বন্ধ)। এই দুইয়ের সমন্বয়ে — transactional integrity + atomic conditional update — উচ্চ concurrency-তেও balance কখনো ভুল বা negative হয় না।
