# ০৬ — KYC Verification Workflow

## Feature-টা আসলে কী?

**KYC (Know Your Customer)** হলো user-এর পরিচয় যাচাই করার প্রক্রিয়া। Agent তার NID number, NID-এর ছবি ও selfie জমা দেয়; admin সেটা দেখে **approve** বা **reject** করে। যতক্ষণ KYC approve না হয়, ততক্ষণ user-এর কিছু সুবিধা সীমিত থাকে।

Bangladesh-এ mobile financial service চালাতে KYC আইনগতভাবে বাধ্যতামূলক (money laundering ঠেকাতে), তাই এটা একটা core compliance feature।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| User UI | `src/app/(app)/settings/kyc/page.tsx` | NID/selfie submit form |
| User API | `src/app/api/kyc/route.ts` | KYC submit ও status দেখা |
| Admin UI | `src/app/admin/kyc/page.tsx` | pending KYC তালিকা |
| Admin API | `src/app/api/admin/kyc/route.ts` (list), `src/app/api/admin/kyc/[id]/route.ts` (approve/reject) | review |
| DB | `Kyc` model | records |

### জড়িত file গুলো সংক্ষেপে

- **`src/app/(app)/settings/kyc/page.tsx`** — user-এর KYC submit form: full name, NID number, NID-এর ছবি ও selfie; current status অনুযায়ী ভিন্ন UI দেখায়।
- **`src/app/api/kyc/route.ts`** — user-এর KYC submit করা ও নিজের KYC status দেখার endpoint।
- **`src/app/admin/kyc/page.tsx`** — admin-এর UI: pending KYC-দের তালিকা, তাদের তথ্য/ছবি দেখা, approve/reject করা।
- **`src/app/api/admin/kyc/route.ts`** — admin-এর জন্য KYC list (filter/pagination) endpoint।
- **`src/app/api/admin/kyc/[id]/route.ts`** — admin-only; একটা নির্দিষ্ট KYC approve/reject করে, `reviewedBy`/`reviewedAt`/`rejectedReason` সহ।
- **`prisma/schema.prisma → Kyc`** — KYC-র মূল record; `userId @unique` (এক user = এক KYC), status, review fields।

## পুরো flow-টা কীভাবে কাজ করে

### ধাপ ১ — User KYC জমা দেয় (status: PENDING)

User full name, NID number, আর ছবিগুলো (data URL হিসেবে) পাঠায়। একটা `Kyc` record তৈরি হয় `status: PENDING` দিয়ে। এক user-এর একটাই KYC থাকে (schema-তে `userId @unique`)।

### ধাপ ২ — Admin review করে (approve/reject)

Admin API-তে প্রথমে role যাচাই, তারপর সিদ্ধান্ত:

```ts
// src/app/api/admin/kyc/[id]/route.ts
const user = await getRequestUser(req);
if (!user || user.role !== "ADMIN") return unauthorized();

const { action, reason } = body;   // "approve" | "reject"
if (action !== "approve" && action !== "reject") return badRequest("Invalid action");

const updated = await db.kyc.update({
  where: { id },
  data: {
    status: action === "approve" ? "APPROVED" : "REJECTED",
    rejectedReason: action === "reject" ? (reason ?? null) : null,
    reviewedAt: new Date(),
    reviewedBy: user.id,
  },
});
```

### ধাপ ৩ — Status State Machine

```
PENDING ──approve──▶ APPROVED
   │
   └──reject──▶ REJECTED  (rejectedReason সহ)
```

Reject করলে একটা কারণ (`rejectedReason`) রাখা হয়, যাতে user জানতে পারে কেন বাতিল হলো এবং আবার ঠিক করে জমা দিতে পারে।

### ধাপ ৪ — KYC status অনুযায়ী UI বদলায়

User-এর KYC page তার current status অনুযায়ী ভিন্ন ভিন্ন কিছু দেখায় — `APPROVED` হলে "Account Verified", `PENDING` হলে "under review", `REJECTED` হলে কারণসহ আবার জমা দেওয়ার সুযোগ।

## গুরুত্বপূর্ণ concept গুলো

- **Compliance feature**: শুধু technical নয়, আইনি প্রয়োজনে বানানো feature।
- **Human-in-the-loop workflow**: automation নয়, একজন মানুষ (admin) যাচাই করে সিদ্ধান্ত নেয়।
- **One-to-one relation**: এক user = এক KYC (`@unique`)।
- **Audit fields**: `reviewedBy`, `reviewedAt` — কে কখন যাচাই করল।

---

## Interview Questions

### ১. KYC feature-টা technically কেন গুরুত্বপূর্ণ, শুধু একটা form তো?

**উত্তর:** দেখতে সাধারণ form মনে হলেও এটা একটা **compliance ও risk-management** feature। আর্থিক নিয়ন্ত্রক সংস্থা (যেমন Bangladesh Bank) mobile financial service-এ user identity যাচাই বাধ্যতামূলক করে, যাতে money laundering, terror financing, ভুয়া account ঠেকানো যায়। Technically এটা একটা multi-step workflow: sensitive data (NID, selfie) সংগ্রহ, নিরাপদে সংরক্ষণ, admin review, status tracking, এবং সিদ্ধান্তের audit। তাই এটা শুধু form নয় — এটা একটা নিয়ন্ত্রিত, auditable প্রক্রিয়া যার সাথে আইনি দায়ও জড়িত।

### ২. Schema-তে `Kyc.userId`-কে `@unique` রাখা হয়েছে কেন?

**উত্তর:** কারণ একজন user-এর একটাই KYC থাকা উচিত — এক ব্যক্তি, এক পরিচয়। `@unique` constraint database-level-এ নিশ্চিত করে যে একই user-এর জন্য দুটো KYC record কখনো তৈরি হতে পারবে না। এটা না থাকলে কেউ একাধিকবার submit করে অনেকগুলো pending KYC বানিয়ে ফেলতে পারত, admin-এর review queue জট পাকাত, আর "এই user-এর আসল KYC কোনটা?" — এই দ্বিধা তৈরি হতো। Database constraint দিয়ে এই business rule enforce করা application-level check-এর চেয়ে নিরাপদ, কারণ এটা race condition-এও ভাঙে না।

### ৩. Reject করার সময় `rejectedReason` রাখার গুরুত্ব কী?

**উত্তর:** `rejectedReason` user-কে **feedback** দেয় — কেন তার KYC বাতিল হলো (যেমন "NID ছবি ঝাপসা", "নাম মেলেনি")। এটা না থাকলে user শুধু "rejected" দেখত, কিন্তু কী ঠিক করবে বুঝত না, ফলে বারবার একই ভুলে জমা দিত এবং admin-এর কাজ বাড়ত। পাশাপাশি এটা admin-এর সিদ্ধান্তের একটা রেকর্ডও — পরে যদি প্রশ্ন ওঠে "এই KYC কেন reject হলো", তার উত্তর রেকর্ডে থাকে। ভালো UX আর accountability — দুটোই এতে সাধিত হয়।

### ৪. এই approve/reject-এ `reviewedBy` ও `reviewedAt` কেন রাখা হয়?

**উত্তর:** এগুলো **audit trail** — কোন admin (`reviewedBy`) কখন (`reviewedAt`) এই KYC-এর সিদ্ধান্ত নিয়েছে তার প্রমাণ। KYC যেহেতু compliance-সম্পর্কিত, পরে regulatory audit বা তদন্তে "কে এই পরিচয় approve করেছিল" জানা জরুরি হতে পারে। যদি কোনো ভুয়া account approve হয়ে যায়, এই field দিয়ে দায়িত্ব নির্ধারণ করা যায়। এটা add-money approval-এর `reviewedBy`-র মতোই একই accountability নীতি — সংবেদনশীল সিদ্ধান্তের সাথে সবসময় "কে নিল, কখন নিল" যুক্ত রাখা।

### ৫. KYC review-টা কেন automation দিয়ে না করে admin (human) দিয়ে করানো হয়েছে?

**উত্তর:** কারণ পরিচয় যাচাই একটা **judgment-নির্ভর** কাজ — NID-এর ছবি আসল কিনা, selfie-র সাথে মেলে কিনা, তথ্য সঙ্গতিপূর্ণ কিনা — এসব সূক্ষ্ম সিদ্ধান্ত। ভুল হলে আর্থিক ও আইনি ঝুঁকি বড়, তাই এখানে **human-in-the-loop** রাখা নিরাপদ। বাস্তব বড় system-এ AI/OCR দিয়ে প্রাথমিক যাচাই হয় ঠিকই, কিন্তু চূড়ান্ত সিদ্ধান্ত সাধারণত মানুষ বা মানুষ+AI মিলে নেয়। এই project-এ সেই workflow-টাই সরল রূপে — admin manually approve/reject করে, আর প্রতিটা সিদ্ধান্ত auditable থাকে।

### ৬. NID/selfie-র ছবি data URL হিসেবে রাখার সমস্যা কী, ভালো বিকল্প কোনটা?

**উত্তর:** এই demo-তে ছবি base64 data URL হিসেবে (সম্ভবত DB text field-এ) রাখা হয়েছে, যা প্রোটোটাইপে সহজ কিন্তু production-এ সমস্যাজনক: (ক) base64 আসল file-এর চেয়ে ~33% বড়, তাই DB ফুলে যায় ও query ধীর হয়; (খ) প্রতিবার row পড়লে বিশাল ছবিও load হয়; (গ) DB backup ভারী হয়ে যায়। ভালো বিকল্প — ছবি একটা **object storage**-এ (AWS S3, Cloudflare R2 ইত্যাদি) রাখা, আর DB-তে শুধু তার URL/key সংরক্ষণ করা। এমনকি sensitive হওয়ায় সেগুলো private bucket-এ রেখে **signed URL** দিয়ে সীমিত-সময়ের access দেওয়া উচিত। এতে DB হালকা থাকে, ছবি CDN দিয়ে দ্রুত আসে, আর access নিয়ন্ত্রণ শক্ত হয়।

### ৭. KYC status অনুযায়ী feature সীমিত করা — এটা কোথায় enforce করা উচিত, frontend না backend?

**উত্তর:** অবশ্যই **backend**-এ (frontend-এ শুধু UX-এর জন্য, নিরাপত্তার জন্য নয়)। যদি কোনো feature (যেমন বড় transfer) শুধু KYC-approved user-এর জন্য হয়, তাহলে সেই নিয়মটা সংশ্লিষ্ট API route-এ প্রয়োগ করতে হবে — request আসার সময় server DB থেকে user-এর KYC status দেখে সিদ্ধান্ত নেবে। শুধু frontend-এ বাটন লুকালে user সরাসরি API-তে request পাঠিয়ে সেটা bypass করতে পারবে (ঠিক admin guard-এর মতোই যুক্তি)। এটা আবারও সেই মূল নীতি — **client-side check শুধু সুবিধার জন্য, business rule ও authorization সবসময় server-side-এ enforce করতে হয়**।

### ৮. `nidNumber`-এর মতো sensitive data কীভাবে protect করা উচিত?

**উত্তর:** NID number একটা অত্যন্ত সংবেদনশীল personal identifier, তাই একাধিক স্তরে সুরক্ষা দরকার: (ক) **encryption at rest** — DB-তে plaintext-এ না রেখে encrypt করে রাখা, যাতে DB leak হলেও সরাসরি পড়া না যায়; (খ) **access control** — শুধু অনুমোদিত admin/endpoint এটা পড়তে পারবে, আর প্রতিটা access audit log-এ থাকবে; (গ) **masking** — UI/log-এ পুরো number না দেখিয়ে আংশিক (`****1234`) দেখানো; (ঘ) transit-এ সবসময় HTTPS। এছাড়া data-minimization নীতি — যতটুকু দরকার ঠিক ততটুকুই সংরক্ষণ, আর নিয়ম অনুযায়ী নির্দিষ্ট সময় পর মুছে ফেলা। PII সুরক্ষা অনেক দেশে আইনগত বাধ্যবাধকতাও।

### ৯. KYC reject হওয়ার পর user আবার submit করলে status flow কীভাবে reset হয়?

**উত্তর:** যেহেতু এক user-এর একটাই `Kyc` row (`userId @unique`), resubmit মানে নতুন row তৈরি নয় — বিদ্যমান row-এর status আবার `PENDING`-এ ফিরিয়ে আনা এবং নতুন তথ্য/ছবি দিয়ে update করা (আগের `rejectedReason` মুছে দিয়ে)। ফলে flow দাঁড়ায়: `REJECTED → (resubmit) → PENDING → APPROVED/REJECTED`। এই approach-এর সুবিধা — এক user-এর KYC-র একটাই বর্তমান অবস্থা থাকে, admin queue-তে ডুপ্লিকেট জমে না। তবে ইতিহাস রাখতে চাইলে (কতবার reject হলো, কেন) একটা আলাদা `KycAttempt`/history table রাখা যেত — কিন্তু current design-এ সরলতার জন্য একটাই mutable record রাখা হয়েছে।

### ১০. `nidNumber` একই সাথে `Kyc` আর `Profile` — দুই model-এ আছে; এই duplication কি সমস্যা?

**উত্তর:** এটা একটা সচেতন trade-off। `Kyc.nidNumber` হলো verification-এর অংশ (যা admin যাচাই করে approve করেছে), আর `Profile.nidNumber` হয়তো user-এর সাধারণ profile-এ দ্রুত দেখানোর জন্য। Duplication-এর ঝুঁকি হলো **data drift** — এক জায়গায় বদলালে অন্য জায়গায় না বদলালে দুটো অসঙ্গত হয়ে যায়, আর "কোনটা সঠিক?" বিভ্রান্তি তৈরি হয়। ভালো practice — একটা **single source of truth** রাখা (যেমন verified NID শুধু `Kyc`-এ), আর অন্য জায়গায় দরকার হলে সেখান থেকে reference/read করা, copy না রাখা। যদি performance-এর জন্য duplicate রাখতেই হয়, তাহলে দুটো সবসময় sync রাখার দায়িত্ব (একই transaction-এ update) স্পষ্টভাবে নিতে হবে।
