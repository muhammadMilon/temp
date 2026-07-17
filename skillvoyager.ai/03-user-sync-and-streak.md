# 03 — User Sync & Streak Engine (Firebase → MongoDB)

## এই ফিচারটা আসলে কী?

Firebase জানে "কে login করেছে" (uid, email, নাম, ছবি) — কিন্তু SkillVoyager-এর নিজের অনেক data দরকার যেটা Firebase-এ রাখা যায় না বা উচিত না: user-এর **points, streak, role, onboarding answers, bookmarks, badges**। এসব থাকে আমাদের নিজের **MongoDB**-তে।

তাহলে সমস্যা হলো — Firebase-এর user আর MongoDB-র user record দুটোকে জোড়া লাগাতে হবে। এই জোড়া লাগানোর কাজটা করে `POST /api/users` endpoint। প্রতিবার user login/refresh করলে frontend এই endpoint-এ hit করে, আর backend `uid` ধরে MongoDB-তে user-কে **upsert** করে (থাকলে update, না থাকলে create)।

এই endpoint-এর ভেতরেই একটা মজার business logic আছে — **streak engine**। অর্থাৎ user কত দিন টানা active — গতকাল এসেছিল আর আজ এলে streak +1, একদিন gap পড়লে streak reset হয়ে 1। এই "consecutive active days" গেমিফিকেশনের একটা মূল উপাদান।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| API | `backend/server.js` (`POST /api/users`) | Firebase user → MongoDB upsert + streak হিসাব |
| DB Model | `backend/models/User.js` | User schema (uid, points, streak, role, onboarding...) |
| Caller | `frontend/src/providers/AuthProvider.jsx` | login/refresh-এ এই endpoint কল করে |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`backend/server.js` → `POST /api/users`** — পুরো sync ও streak logic এখানে; Firebase identity-কে আমাদের DB record-এ পরিণত করে এবং role assign করে।
- **`backend/models/User.js`** — Mongoose schema যেটা user-এর সব field (`uid` unique, `streak`, `lastStreakDate`, `points`, `role` enum, nested `onboarding`/`profile`) সংজ্ঞায়িত করে।
- **`frontend/src/providers/AuthProvider.jsx`** — `onAuthStateChanged`-এর ভেতর থেকে এই endpoint-এ POST করে, ফেরত পাওয়া `user` (role সহ) `dbUser`-এ রাখে।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — User schema-তে streak field

`User.js`-এ streak track করার জন্য দুটো field জরুরি — কত দিনের streak, আর সর্বশেষ কবে streak update হয়েছিল।

```js
// backend/models/User.js
const userSchema = new mongoose.Schema({
    uid: { type: String, required: true, unique: true },   // ⬅️ Firebase uid, unique key
    role: { type: String, enum: ['user', 'admin'], default: 'user' },
    points: { type: Number, default: 0 },
    streak: { type: Number, default: 0 },            // consecutive active days
    lastStreakDate: { type: Date, default: null },   // last time streak was bumped
    // ... onboarding, profile, bookmarks, badges ইত্যাদি
}, { timestamps: true });
```

`unique: true` on `uid` নিশ্চিত করে এক Firebase user = ঠিক এক MongoDB record।

### ধাপ ২ — DB connected কিনা আগে চেক

Endpoint শুরুতেই দেখে MongoDB connected কিনা — না থাকলে login যেন না ভাঙে, তাই graceful response দেয় (crash না করে)।

```js
// backend/server.js → POST /api/users
if (mongoose.connection.readyState !== 1) {
    return res.status(200).json({ success: false, message: 'Database not connected' });
}
const { uid, email, displayName, photoURL, institute } = req.body || {};
if (!uid) return res.status(400).json({ success: false, message: 'uid is required' });
```

### ধাপ ৩ — Streak হিসাব (দিনের পার্থক্য)

এই অংশটাই engine। আজকের তারিখকে midnight-এ normalize করা হয় (সময় বাদ), তারপর `lastStreakDate`-এর সাথে পার্থক্য (দিনে) হিসাব করা হয়।

```js
// backend/server.js → POST /api/users
const today = new Date();
today.setHours(0, 0, 0, 0);   // ⬅️ সময় বাদ, শুধু তারিখ

if (existingUser) {
    let lastStreakDate = existingUser.lastStreakDate;
    if (lastStreakDate) {
        lastStreakDate = new Date(lastStreakDate);
        lastStreakDate.setHours(0, 0, 0, 0);
        const diffDays = Math.round((today - lastStreakDate) / (1000 * 60 * 60 * 24));

        if (diffDays === 1) {                          // গতকাল এসেছিল → streak বাড়ো
            streakUpdate.streak = (existingUser.streak || 0) + 1;
            streakUpdate.lastStreakDate = today;
        } else if (diffDays > 1) {                     // gap পড়েছে → reset
            streakUpdate.streak = 1;
            streakUpdate.lastStreakDate = today;
        } else if (diffDays === 0) {                   // আজকেই আগে এসেছে → অপরিবর্তিত
            if (!existingUser.streak) { streakUpdate.streak = 1; streakUpdate.lastStreakDate = today; }
        }
    } else { streakUpdate.streak = 1; streakUpdate.lastStreakDate = today; }
} else { streakUpdate.streak = 1; streakUpdate.lastStreakDate = today; }   // নতুন user
```

মূল বুদ্ধি: `setHours(0,0,0,0)` — সময় বাদ দিয়ে শুধু "কয়টা ক্যালেন্ডার দিন পার হয়েছে" মাপা, নাহলে একই দিনে সকাল-সন্ধ্যা login-ও আলাদা মনে হতো।

### ধাপ ৪ — Upsert (atomic create-or-update)

সব field একটা `update` object-এ জমা করে `findOneAndUpdate` দিয়ে upsert করা হয়। খেয়াল করুন conditional spread — যে field পাঠানো হয়নি সেটা DB-তে touch হয় না।

```js
// backend/server.js → POST /api/users
const update = {
    lastLoginAt: new Date(),
    ...streakUpdate,
    ...(email != null && { email: String(email) }),
    ...(displayName != null && { displayName: String(displayName) }),
    ...(email === 'admin@skillvoyager.ai' && { role: 'admin' }),   // ⬅️ admin role assign
};
const user = await User.findOneAndUpdate(
    { uid: String(uid) },
    { $set: update },
    { new: true, upsert: true, runValidators: false }   // ⬅️ না থাকলে বানাও, নতুন doc ফেরত দাও
);
res.status(200).json({ success: true, user });
```

`upsert: true` = নতুন হলে create; `new: true` = update-এর পরের version ফেরত; `$set` = শুধু দেওয়া field বদলাও।

## মূল ধারণা (Key Concepts)

- **Identity vs. domain data** — Firebase "কে" রাখে, MongoDB "সে আমাদের কাছে কী" রাখে; দুটোকে `uid` দিয়ে জোড়া।
- **Upsert** — "update else insert" এক atomic operation-এ; আলাদা `find` তারপর `create` করতে হয় না (race condition কম)।
- **Date normalization** — `setHours(0,0,0,0)` দিয়ে time বাদ দিয়ে শুধু calendar-day তুলনা।
- **Conditional spread (`...(cond && {...})`)** — শুধু উপস্থিত field update করা, বাকিটা অক্ষত রাখা।
- **Graceful degradation** — DB down থাকলেও login না ভেঙে ভদ্রভাবে response দেওয়া।
- **Idempotent sync** — একই login বারবার হলেও DB-তে duplicate তৈরি হয় না।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. `upsert` ব্যবহার করলেন কেন, আগে `findOne` করে না পেলে `create` করার বদলে?
**উত্তর:** `findOne` তারপর `create` — এই দুই ধাপে একটা **race condition** থাকে: user দ্রুত দুটো tab খুললে দুটো request প্রায় একসাথে আসতে পারে, দুটোই `findOne`-এ "নেই" পেয়ে দুটো create করবে — duplicate record বা unique-index error। `findOneAndUpdate(..., { upsert: true })` পুরো কাজটা MongoDB-র একটা atomic operation-এ করে, তাই concurrent request থাকলেও একটাই doc তৈরি/আপডেট হয়। এটা কম কোড, কম bug, এবং database-এর concurrency guarantee ব্যবহার করে।

### ২. Streak হিসাবে `setHours(0,0,0,0)` না করলে কী সমস্যা হতো?
**উত্তর:** সময়সহ তুলনা করলে "diff in days" ভুল হতো। ধরুন user আজ রাত ৯টায় এসেছিল, পরদিন সকাল ৮টায় এলো — সময়সহ পার্থক্য ২৩ ঘণ্টা, `Math.round(diff)` হয়তো ১ দিন দেখাত, কিন্তু edge case-এ ভুল হতে পারত। উল্টো দিকে একই দিনে সকাল-বিকাল দুবার এলে সময়সহ diff শূন্যের বেশি হয়ে streak ভুলভাবে বাড়ত। `setHours(0,0,0,0)` দুই তারিখকেই midnight-এ এনে দেয়, ফলে তুলনা হয় বিশুদ্ধ "কয়টা ক্যালেন্ডার দিন পার হয়েছে" — এটাই streak-এর সঠিক সংজ্ঞা।

### ৩. এই streak logic timezone নিয়ে কী সমস্যায় পড়তে পারে?
**উত্তর:** বড় সমস্যা — `new Date()` সার্ভারের timezone (Vercel-এ সাধারণত UTC) ব্যবহার করে, কিন্তু user হয়তো বাংলাদেশে (UTC+6)। ফলে user-এর কাছে যখন রাত ১১টা (এখনো "আজ"), সার্ভারে তখন পরদিন হয়ে গেছে — user-এর মনে হবে একই দিনে এসেছে, সিস্টেম ভাববে নতুন দিন, বা উল্টো। এতে streak কারো কাছে অন্যায্যভাবে ভাঙতে/বাড়তে পারে। ঠিক করতে হলে user-এর timezone সংরক্ষণ করে সেই অনুযায়ী "day boundary" হিসাব করা উচিত, বা অন্তত পুরো সিস্টেমে একটা নির্দিষ্ট timezone (যেমন Asia/Dhaka) ঠিক করে নেওয়া। এখনকার implementation demo-পর্যায়ে ঠিক আছে, production-এ এটা fix করার মতো।

### ৪. Streak logic কেন login endpoint-এ, আলাদা service-এ না? এটা ভালো design?
**উত্তর:** সততার সাথে — এটা আদর্শ না। এখন streak-এর প্রায় একই logic তিন জায়গায় ছড়িয়ে আছে: `POST /api/users` (login), `leaderboard.routes` (`add-points`), আর `roadmap.controller` (roadmap বানালে)। এটা DRY ভাঙে — একটা জায়গায় বদলালে বাকিগুলো ভুলে যাওয়ার ঝুঁকি। ভালো হতো একটা `updateStreak(user)` utility/service বানিয়ে সব জায়গা থেকে সেটা কল করা। Login endpoint-এ এটা রাখার কারণ ছিল সুবিধা — প্রতি login-এ activity ধরা পড়ে; কিন্তু logic-টা reusable function-এ তুলে আনলে maintainable হতো।

### ৫. `runValidators: false` কেন দিলেন, এতে কী ঝুঁকি?
**উত্তর:** `findOneAndUpdate`-এ default-এ Mongoose schema validator চালায় না; এখানে explicit `false` দেওয়া মানে partial update-এ validation skip করা যাতে কিছু field না থাকলেও update আটকে না যায়। ঝুঁকি হলো — অবৈধ data (যেমন ভুল enum, missing required) DB-তে ঢুকে যেতে পারে যা পরে অন্য জায়গায় সমস্যা করবে। যেহেতু এখানে আমরা নিজেরাই নিয়ন্ত্রিতভাবে field সেট করছি (user input সরাসরি না), ঝুঁকি সীমিত; তবু ভালো practice হলো `runValidators: true` রেখে schema-কে single source of truth করা। এটা একটা সচেতন trade-off — flexibility বনাম integrity।

### ৬. `...(email === 'admin@skillvoyager.ai' && { role: 'admin' })` — এই role assign কতটা নিরাপদ?
**উত্তর:** অনিরাপদ। এখানে role assign হচ্ছে client-এর পাঠানো `email`-এর ভিত্তিতে, অথচ সেই email সার্ভারে verify করা হয় না — কেউ চাইলে `POST /api/users`-এ যেকোনো `uid` দিয়ে `email: 'admin@skillvoyager.ai'` পাঠিয়ে নিজেকে admin বানাতে পারে। সঠিক approach হতো Firebase ID token verify করে token থেকে verified email/uid নেওয়া, এবং admin list backend-এ (env var বা DB) trusted জায়গায় রাখা — client-এর কথায় role দেওয়া নয়। এটা এই সিস্টেমের privilege-escalation দুর্বলতা, hardening-এর তালিকায় উপরে।

### ৭. Login-এর জন্য এই backend sync fail করলে কী হয়? সেটা কি intentional?
**উত্তর:** হ্যাঁ, intentional এবং ভালো decision। `AuthProvider`-এর `saveUserToDb`/observer-এ backend call `try/catch`-এ মোড়া এবং fail করলে শুধু `console.warn` করে — login block করে না। কারণ authentication-এর authority Firebase, আমাদের DB sync একটা secondary concern (points/role আনা)। DB down থাকলে বা network সমস্যা হলেও user যেন অ্যাপে ঢুকতে পারে — এটাই graceful degradation। endpoint নিজেও `readyState !== 1` হলে 200 দিয়ে ভদ্রভাবে জানায় "DB connected না", 500 দিয়ে ভাঙে না। Trade-off: তখন `dbUser` (role) আসবে না, তাই admin feature সাময়িক কাজ করবে না — যা গ্রহণযোগ্য।

### ৮. `$set` ছাড়া পুরো object দিয়ে update করলে কী হতো?
**উত্তর:** MongoDB-তে update object-এ যদি `$set`-এর মতো operator না থাকে (শুধু plain field), তাহলে সেটা **document replace** করে — অর্থাৎ যে field গুলো object-এ নেই সেগুলো মুছে যায়। এখানে যদি `$set` না দিয়ে সরাসরি `{ email, streak, ... }` দিতাম, তাহলে user-এর points, bookmarks, badges, onboarding — যা এই update-এ নেই — সব উড়ে যেত। `$set` নিশ্চিত করে শুধু উল্লেখিত field বদলায়, বাকি সব অক্ষত থাকে। এটা partial update-এর জন্য অপরিহার্য।

### ৯. একই login বারবার হলে (refresh, multi-tab) এই design কীভাবে সামলায়?
**উত্তর:** এটা idempotent-এর কাছাকাছি ভাবে ডিজাইন করা। `uid` unique এবং upsert ব্যবহার করায় বারবার call হলেও একটাই record থাকে — duplicate হয় না। Streak logic-ও safe: একই দিনে বারবার login হলে `diffDays === 0` হয়ে streak অপরিবর্তিত থাকে, শুধু `lastLoginAt` update হয়। তাই multi-tab বা ঘন ঘন refresh streak-কে ভুলভাবে বাড়ায় না। একমাত্র সূক্ষ্ম ঝুঁকি concurrent প্রথম-বার-create, যেটা upsert-এর atomicity + unique index মিলে সামলায় (একটা সফল হবে, অন্যটা duplicate-key-এ ব্যর্থ হয়ে retry/update হবে)।

### ১০. এই "streak on login" পদ্ধতির business দুর্বলতা কী? কীভাবে উন্নত করতেন?
**উত্তর:** দুর্বলতা হলো — শুধু login করলেই streak বাড়ে, কোনো শেখার কাজ না করলেও। User প্রতিদিন এসে সাথে সাথে বেরিয়ে গেলেও streak বাড়বে, যা gamification-এর মূল উদ্দেশ্য (নিয়মিত **শেখা**) কে দুর্বল করে — এটা vanity metric হয়ে যেতে পারে। উন্নত করতে streak-কে অর্থবহ activity-র সাথে বাঁধতাম: quiz সম্পন্ন করা, milestone শেষ করা, বা নির্দিষ্ট study time — অন্তত একটা meaningful action হলে সেই দিন "active" গণ্য হতো। কোডে ইতিমধ্যে quiz/roadmap সম্পন্ন করলে points ও streak update-এর অংশ আছে, তাই streak-কে সম্পূর্ণভাবে login থেকে সরিয়ে activity-ভিত্তিক করাই স্বাভাবিক পরবর্তী ধাপ।
