# 10 — Stripe Course Payments & Enrollment

## এই ফিচারটা আসলে কী?

SkillVoyager-এ কিছু course paid — user টাকা দিয়ে enroll করতে পারে। টাকার লেনদেন সরাসরি নিজে হ্যান্ডল করা বিপজ্জনক ও legally জটিল (card number, PCI compliance), তাই এটা করা হয়েছে **Stripe** দিয়ে — বিশ্বের সবচেয়ে ব্যবহৃত payment platform।

এখানে ব্যবহার করা হয়েছে **Stripe Checkout Session** — backend একটা session বানায়, user-কে Stripe-এর নিজস্ব hosted payment page-এ redirect করা হয় (card details আমাদের সার্ভারে কখনো আসে না), payment সফল হলে user আমাদের success URL-এ ফিরে আসে। তারপর backend Stripe থেকে session verify করে DB-তে একটা **Enrollment** record বানায়।

এই flow-এর সবচেয়ে গুরুত্বপূর্ণ concern — **idempotency**: একই payment দুবার process হয়ে দুটো enrollment যেন না বানায়। সেটা ঠেকানো হয়েছে `transactionId` (Stripe-এর `payment_intent`)-কে **unique** রেখে ও duplicate check দিয়ে।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| API | `backend/server.js` (Stripe endpoints) | checkout session বানায়, payment verify করে, enrollment save করে |
| DB Model | `backend/models/Enrollment.js` | enrollment record (`transactionId` unique) |
| UI | `frontend/src/pages.jsx/Courses.jsx` | course list, "Enroll" → checkout |
| UI | `frontend/src/pages.jsx/CoursePaymentSuccess.jsx` | Stripe থেকে ফিরে এসে verify করে |
| Config | `frontend/package.json` (`@stripe/stripe-js`) | Stripe client SDK |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`backend/server.js`** — Stripe-এর সব endpoint এখানে: `create-course-checkout-session`, `course-payment-success`, `verify-session`, `enrollment/verify`; Stripe secret key দিয়ে server-side এ কাজ করে।
- **`Enrollment.js`** — enrollment schema; `transactionId` unique — এটাই idempotency-র চাবি।
- **`CoursePaymentSuccess.jsx`** — payment-এর পর Stripe redirect করা page; session verify করে enrollment নিশ্চিত করে।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — Server-side এ Stripe initialize (secret key)

Stripe-এর secret key **কখনো frontend-এ যায় না** — শুধু backend-এ, env var থেকে। এটাই টাকার নিরাপত্তার প্রথম নিয়ম।

```js
// backend/server.js
const Stripe = require('stripe');
const stripe = Stripe(process.env.STRIPE_SECRET_KEY);   // ⬅️ server-only secret
```

### ধাপ ২ — Checkout Session বানানো

Frontend "Enroll" চাপলে backend একটা Checkout Session বানায় — কত টাকা, কোন course, সফল হলে কোথায় ফিরবে (success_url), আর `metadata`-তে courseId/userEmail (পরে verify-এর সময় লাগবে)।

```js
// backend/server.js  (POST /api/create-course-checkout-session)
const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    mode: 'payment',
    line_items: [{
        price_data: {
            currency: currency || 'bdt',
            product_data: { name: courseTitle || 'SkillVoyager Course', description: '... Lifetime Access' },
            unit_amount: amount,      // ⬅️ frontend থেকে price * 100 (সেন্ট/পয়সা)
        },
        quantity: 1,
    }],
    success_url: finalSuccessUrl,     // courseId + {CHECKOUT_SESSION_ID} সহ
    cancel_url: `${clientUrl}/courses`,
    metadata: { courseId: String(courseId||''), courseTitle: String(courseTitle||''), userEmail: String(userEmail||'') },
});
res.json({ url: session.url });       // ⬅️ frontend এই URL-এ redirect করবে
```

`unit_amount` সবসময় ক্ষুদ্রতম একক-এ (৳১০০ = `10000` পয়সা) — Stripe-এর নিয়ম, floating-point ভুল এড়াতে।

### ধাপ ৩ — User Stripe-এর page-এ যায় ও ফিরে আসে

Frontend `session.url`-এ redirect করে; user Stripe-এর hosted page-এ card দিয়ে pay করে (card আমাদের কাছে আসে না); সফল হলে `success_url`-এ ফিরে আসে, যেখানে `{CHECKOUT_SESSION_ID}` Stripe-এর আসল session id দিয়ে replace হয়ে যায়।

### ধাপ ৪ — ফিরে এসে payment verify (client-কে বিশ্বাস না করা)

User ফিরলেই "paid" ধরে নেওয়া যায় না — কেউ manually success URL খুলতে পারে। তাই backend Stripe থেকে session **retrieve** করে সত্যিই `payment_status === 'paid'` কিনা যাচাই করে।

```js
// backend/server.js  (GET /api/course-payment-success)
const session = await stripe.checkout.sessions.retrieve(session_id, { expand: ['line_items'] });
if (session.payment_status !== 'paid') {
    return res.status(400).json({ success: false, message: 'Payment not completed' });   // ⬅️ verify
}
```

### ধাপ ৫ — Idempotency: duplicate enrollment ঠেকানো

Verify হলে enrollment বানানোর আগে দেখা হয় এই `payment_intent` দিয়ে আগে enrollment হয়েছে কিনা — কারণ user refresh করলে বা দুবার এলে এই endpoint দুবার চলতে পারে।

```js
// backend/server.js  (GET /api/course-payment-success)
const alreadyEnrolled = await Enrollment.findOne({ transactionId: session.payment_intent });
if (alreadyEnrolled) {
    return res.json({ success: true, message: 'Already enrolled', courseId: session.metadata.courseId });
}
await Enrollment.create({
    courseId: session.metadata.courseId || courseId || '',
    courseTitle: session.metadata.courseTitle || '',
    userEmail: session.metadata.userEmail || 'guest@skillvoyager.ai',
    amountPaid: session.amount_total / 100,       // পয়সা → টাকা
    paymentStatus: 'paid',
    transactionId: session.payment_intent,        // ⬅️ unique — দ্বিতীয়বার create ব্যর্থ হবে
    enrolledAt: new Date(),
});
```

### ধাপ ৬ — Schema-স্তরে unique guarantee

Application-এর duplicate check-এর পাশাপাশি DB-স্তরেও `transactionId` unique — দুই layer সুরক্ষা।

```js
// backend/models/Enrollment.js
const enrollmentSchema = new mongoose.Schema({
    courseId:      { type: String, required: true },
    userEmail:     { type: String, required: true },
    amountPaid:    { type: Number },
    paymentStatus: { type: String, default: 'paid' },
    transactionId: { type: String, unique: true },   // ⬅️ DB enforce করে idempotency
    enrolledAt:    { type: Date, default: Date.now },
});
```

## মূল ধারণা (Key Concepts)

- **Hosted checkout (PCI offloading)** — card details Stripe সামলায়, আমাদের সার্ভারে আসে না।
- **Secret vs. publishable key** — secret শুধু server-এ, কখনো client-এ না।
- **Server-side verification** — redirect-এর পর "paid" client-এর কথায় না, Stripe থেকে যাচাই করে।
- **Idempotency** — একই payment বারবার এলেও ঠিক একটা enrollment; `transactionId` unique।
- **Smallest currency unit** — টাকা integer পয়সায় (×১০০), floating-point ভুল এড়াতে।
- **Metadata pass-through** — courseId/email session-এ রেখে পরে verify-এ ব্যবহার।
- **Defense in depth** — app-স্তরে duplicate check + DB-স্তরে unique index।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. Stripe Checkout (hosted) ব্যবহার করলেন কেন, নিজের payment form বানিয়ে card নেওয়ার বদলে?
**উত্তর:** মূলত নিরাপত্তা ও compliance-এর জন্য। কেউ যদি নিজের সার্ভারে raw card number নেয়, তাহলে সে **PCI DSS** compliance-এর আওতায় পড়ে — যা অত্যন্ত কঠোর, ব্যয়বহুল ও audit-নির্ভর; একটা ছোট প্রোজেক্টের পক্ষে অবাস্তব। Stripe Checkout-এ card details সরাসরি Stripe-এর hosted page-এ যায়, আমাদের সার্ভার কখনো card number স্পর্শ করে না — ফলে আমরা PCI scope-এর বাইরে থাকি, breach-এর দায় কমে। বোনাস — Stripe fraud detection, 3D Secure, multiple payment method, mobile-optimized UI সব দেয়। Trade-off: UI-এর উপর কম নিয়ন্ত্রণ (Stripe-এর page), আর Stripe-এর উপর নির্ভরতা; কিন্তু payment-এর ক্ষেত্রে এই trade-off প্রায় সবসময় যুক্তিসঙ্গত।

### ২. User success URL-এ ফিরলেই enrollment না দিয়ে Stripe থেকে verify করলেন কেন?
**উত্তর:** কারণ success URL client-controlled — user (বা যে কেউ) সরাসরি `/course-payment-success?...` URL browser-এ টাইপ করে খুলতে পারে, আসলে কোনো টাকা না দিয়ে। যদি ফিরে আসা মানেই "paid" ধরতাম, তাহলে বিনামূল্যে course unlock করা যেত — সরাসরি জালিয়াতি। তাই backend Stripe API থেকে `sessions.retrieve` করে **Stripe-এর** কাছে জিজ্ঞেস করে "এই session কি সত্যিই paid?" — এটাই একমাত্র authoritative উৎস। এটা আবারও "never trust the client" নীতি: টাকার মতো গুরুত্বপূর্ণ কিছুতে redirect/URL-এর উপর নির্ভর না করে payment provider-এর সাথে সরাসরি যাচাই করা বাধ্যতামূলক।

### ৩. Idempotency ঠিক কী সমস্যা সমাধান করে এখানে? না থাকলে কী ঘটত?
**উত্তর:** Idempotency নিশ্চিত করে একই payment যতবারই process হোক, ফলাফল একটাই enrollment। বাস্তবে এই endpoint একাধিকবার চলতে পারে — user success page refresh করল, back-forward করল, বা network retry হলো, বা code-এ একাধিক verify path (GET + PATCH + verify-session) আছে যা একই payment হ্যান্ডল করে। idempotency না থাকলে প্রতিবার একটা নতুন Enrollment তৈরি হতো — user একবার টাকা দিয়ে DB-তে ৩-৪টা enrollment record, ভুল analytics, ভুল transaction list, হয়তো ভুল "কতবার কিনল" গণনা। এখানে `transactionId` (Stripe payment_intent) unique রেখে ও create-এর আগে `findOne` করে এটা ঠেকানো হয়েছে — একই payment_intent-এ দ্বিতীয়বার এলে "Already enrolled" ফেরত যায়, নতুন record হয় না।

### ৪. `transactionId` app-স্তরে check করছেন, আবার DB-তে unique-ও করেছেন — দুটোই কেন? একটা যথেষ্ট না?
**উত্তর:** এটা defense in depth, এবং দুটোর ভূমিকা আলাদা। App-স্তরের `findOne` check দেয় ভালো UX — duplicate হলে graceful "Already enrolled" message, error না। কিন্তু app check একা যথেষ্ট না, কারণ **race condition**: দুটো request প্রায় একসাথে এলে দুটোই `findOne`-এ "নেই" পেয়ে দুটোই create করতে যাবে। এই মুহূর্তে DB-র `unique` index শেষ রক্ষা — দ্বিতীয় `create` duplicate-key error দিয়ে ব্যর্থ হবে, ফলে duplicate DB-তে ঢুকতে পারবে না। অর্থাৎ app check = সাধারণ ক্ষেত্রে সুন্দর behavior, DB unique = concurrency-তেও guarantee। দুটো মিলে correctness (DB) + UX (app) দুটোই দেয়। এটা payment-এ correctness-critical বলেই দুই স্তর।

### ৫. Amount `unit_amount`-এ ×১০০ (পয়সা) পাঠানো হয় — এটা কেন গুরুত্বপূর্ণ?
**উত্তর:** কারণ money নিয়ে floating-point বিপজ্জনক। `0.1 + 0.2 !== 0.3` — floating point-এ দশমিক নিখুঁত না, তাই টাকা float হিসেবে রাখলে rounding error-এ পয়সা হারায়/বাড়ে, যা আর্থিক হিসাবে অগ্রহণযোগ্য। Stripe (এবং সব ভালো payment system) তাই amount নেয় **integer, smallest currency unit-এ** — USD-তে cent, BDT-তে পয়সা — যেমন ৳১০০ = `10000`। Integer arithmetic নিখুঁত, কোনো rounding drift নেই। Frontend তাই `price * 100` পাঠায়, আর enrollment save করার সময় `amount_total / 100` করে আবার টাকায় ফেরানো হয় প্রদর্শনের জন্য। এই convention না মানলে (float পাঠালে) Stripe-ই reject করত বা ভুল amount charge হতো।

### ৬. এই কোডে Stripe **webhook** নেই, redirect-নির্ভর verification। এর দুর্বলতা কী, webhook কীভাবে ভালো করত?
**উত্তর:** এটা এই implementation-এর সবচেয়ে বড় দুর্বলতা। এখন enrollment নির্ভর করে user-এর success page-এ **ফিরে আসার** উপর — কিন্তু user payment করার পর যদি browser বন্ধ করে দেয়, network কেটে যায়, বা redirect fail করে, তাহলে টাকা কাটবে কিন্তু enrollment হবে না — user টাকা দিয়ে course পাবে না (বাজে অভিজ্ঞতা ও support burden)। Stripe **webhook** এই সমস্যা সমাধান করে: Stripe সার্ভার-টু-সার্ভার একটা `checkout.session.completed` event পাঠায় আমাদের backend-এ, user-এর browser নির্বিশেষে — তাই payment সফল হলে enrollment নিশ্চিতভাবে হবে। Webhook signature দিয়ে verify করা হয় (`stripe.webhooks.constructEvent`), তাই এটা authoritative ও নিরাপদ। Production-এ redirect-কে শুধু UX (user-কে "সফল" দেখানো) হিসেবে রেখে, actual enrollment webhook-এ করা উচিত। Redirect + webhook একসাথে থাকলে সবচেয়ে নির্ভরযোগ্য।

### ৭. `server.js`-এ `const Enrollment` দুবার require করা হয়েছে (line ~36 ও ~39) — এটা কী সমস্যা?
**উত্তর:** এটা একটা বাস্তব bug — সততার সাথে বলছি। JavaScript-এ একই scope-এ `const` দিয়ে একই নাম দুবার declare করলে `SyntaxError: Identifier 'Enrollment' has already been declared` হয়, যা module load হওয়ার সময়ই পুরো সার্ভার crash করাতে পারে। এটা সম্ভবত copy-paste/merge-এর ভুল (একটা import আর একটা "NEW" comment সহ import পাশাপাশি রয়ে গেছে)। যদি এই অবস্থায় deploy চলে, তার মানে হয় কোনো build/bundler duplicate সরিয়ে দিচ্ছে, নাহলে এটা নীরব সমস্যা যা যেকোনো মুহূর্তে ভাঙতে পারে। ঠিক করতে — দুটোর একটা `const Enrollment = require('./models/Enrollment')` রেখে অন্যটা মুছে ফেলা উচিত। এটা code review/linter (যেমন ESLint `no-redeclare`) থাকলে সাথে সাথে ধরা পড়ত — যা এই প্রোজেক্টে backend-এ নেই।

### ৮. Payment/enrollment endpoint-এ user identification `userEmail` দিয়ে, আর fallback `'guest@skillvoyager.ai'` — এতে সমস্যা কী?
**উত্তর:** কয়েকটা সমস্যা। প্রথমত, `userEmail` client/metadata থেকে আসে, verify করা হয় না — তাই enrollment কোন account-এর সেটা spoof করা যায় বা ভুল হতে পারে; কেউ অন্যের email দিয়ে enroll দেখাতে পারে। দ্বিতীয়ত, email না থাকলে `'guest@skillvoyager.ai'`-এ পড়ে — মানে সব guest purchase একই fake account-এ জমা হয়, ফলে "কে কী কিনেছে" আলাদা করা যায় না, আর সেই user পরে login করলে তার কেনা course খুঁজে পাবে না (enrollment তার আসল account-এর সাথে যুক্ত না)। সঠিক approach — checkout শুরু করার আগে user-কে login বাধ্যতামূলক করা, এবং enrollment-কে verified `uid`-এর সাথে বাঁধা (email নয়, যা বদলাতে পারে)। তখন course access reliably সেই user-এর সাথে যুক্ত থাকত।

### ৯. Enrollment হওয়ার পর user কীভাবে course access পায়? কোডে কী দেখলেন, এর ঝুঁকি কী?
**উত্তর:** কোডের comment ও flow অনুযায়ী — success-এর পর frontend `courseId` URL থেকে পড়ে **localStorage**-এ save করে course auto-unlock করে, আর `enrollment/verify` endpoint দিয়ে DB-তে enrollment আছে কিনা check করে। ঝুঁকি হলো — localStorage client-side, user নিজে সেখানে `courseId` বসিয়ে দিলে টাকা না দিয়েও frontend-এ course unlock দেখাতে পারে; অর্থাৎ localStorage-নির্ভর unlock নিরাপত্তা না, শুধু সুবিধা। আসল enforcement হতে হবে server-side — প্রতিবার course content দেওয়ার আগে backend যাচাই করবে এই user-এর জন্য paid `Enrollment` আছে কিনা (verified uid দিয়ে)। `enrollment/verify` endpoint-টা সেই দিকে একটা পদক্ষেপ, কিন্তু সেটাও `userEmail` optional ও unverified, তাই এখনো ফাঁক আছে। Content-serving-এর সময় server-side entitlement check-ই একমাত্র নির্ভরযোগ্য gate।

### ১০. এই payment feature production-এ নিতে হলে আপনার top-৩ hardening কী?
**উত্তর:** (১) **Webhook দিয়ে enrollment** — redirect-এর উপর নির্ভরতা সরিয়ে `checkout.session.completed` webhook (signature-verified) দিয়ে enrollment করা, যাতে browser বন্ধ হলেও paid user কখনো course না হারায় — এটা correctness ও trust-এর জন্য সবচেয়ে জরুরি। (২) **Authenticated, server-side entitlement** — checkout ও access দুটোতেই verified Firebase uid ব্যবহার করা; course content দেওয়ার আগে backend enrollment যাচাই করা, localStorage-এর উপর না; guest fallback বাদ দেওয়া। (৩) **Endpoint hardening ও data integrity** — Stripe secret/webhook secret env-এ, amount/currency backend-এ validate করা (client-এর পাঠানো amount বিশ্বাস না করে course-এর dabbase দাম ব্যবহার করা — নাহলে user amount=1 পাঠিয়ে ১ পয়সায় কিনতে পারে!), duplicate `const Enrollment` bug ঠিক করা, আর refund/failed-payment case handle করা। এই তিনটা করলে flow-টা demo থেকে সত্যিকারের production-grade payment system হয়। — বিশেষভাবে (৩)-এর amount-validation একটা critical point: এখন `unit_amount` frontend থেকে আসে, তাই দাম client-side manipulate করা সম্ভব, যা অবশ্যই backend-এ course record থেকে নিতে হবে।
