# 06 — Gamification & Leaderboard System

## এই ফিচারটা আসলে কী?

শেখা ধরে রাখা কঠিন — মানুষ শুরু করে, তারপর ছেড়ে দেয়। এই সমস্যার একটা প্রমাণিত সমাধান হলো **gamification** — points (XP), streak, tier/rank, leaderboard দিয়ে শেখাকে খেলার মতো করা। SkillVoyager-এ user quiz দিলে, roadmap বানালে, milestone শেষ করলে **points** পায়; কত দিন টানা active সেটা **streak**; আর সব user-এর মধ্যে নিজের অবস্থান দেখা যায় **Leaderboard**-এ।

Leaderboard-টা শুধু "কে কত points" এর তালিকা না — এতে আছে **tier** (Rookie → Rising → Expert → Master → Legend), **rank change** (আগের বার থেকে উঠল না নামল), আর **all-time / weekly / monthly** filter। প্রতি সপ্তাহে/মাসে points reset করার endpoint-ও আছে (cron দিয়ে চালানোর জন্য)।

এই পুরো XP অর্থনীতিটা `User` model-এর কয়েকটা field-এ (`points`, `weeklyPoints`, `monthlyPoints`, `streak`, `previousRank`) বসে, আর `leaderboard.routes.js` এটাকে ranking, tier ও filter-এ রূপ দেয়।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| API | `backend/routes/leaderboard.routes.js` | ranking, tier, filter, add-points, weekly/monthly reset |
| DB Model | `backend/models/User.js` | points/weeklyPoints/monthlyPoints/streak/previousRank |
| Producer | `backend/controllers/quiz.controller.js` | quiz দিলে `$inc points` |
| Producer | `backend/controllers/roadmap.controller.js` | roadmap বানালে `$inc points/experience +50` |
| UI | `frontend/src/pages/Leaderboard/Leaderboard.jsx` | ranking table দেখায় |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`leaderboard.routes.js`** — সব ranking logic: sort (filter অনুযায়ী), tier label, rank-change হিসাব, `add-points` (streak সহ), আর reset endpoint।
- **`User.js`** — XP-এর সব counter এখানে (`points`, `weeklyPoints`, `monthlyPoints`, `streak`, `previousRank`, `experience`)।
- **`quiz.controller.js` / `roadmap.controller.js`** — points-এর "producer": নির্দিষ্ট action-এ `$inc` দিয়ে points বাড়ায়।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — Points-এর producer (কোথা থেকে XP আসে)

XP কোনো এক জায়গায় বাড়ে না — বিভিন্ন meaningful action XP দেয়। যেমন roadmap বানালে +৫০, quiz-এ প্রতি সঠিক উত্তরে +১০। `$inc` operator দিয়ে atomic increment।

```js
// backend/controllers/roadmap.controller.js  (roadmap বানানোর পর)
await User.findOneAndUpdate({ email: userEmail }, {
    $inc: { points: 50, weeklyPoints: 50, monthlyPoints: 50, experience: 50 },
    $set: { streak: newStreak, lastStreakDate: today }
});
```

```js
// backend/controllers/quiz.controller.js  (quiz evaluate-এর পর)
const pointsEarned = score * 10;
await User.findOneAndUpdate({ email: userEmail }, { $inc: { points: pointsEarned } }, { new: true });
```

লক্ষ্য করুন: একই action-এ `points`, `weeklyPoints`, `monthlyPoints` তিনটাই বাড়ানো হয় — কারণ তিনটা leaderboard filter আলাদা counter থেকে চলে।

### ধাপ ২ — Filter অনুযায়ী sort

Leaderboard তিন mode-এ চলে — all-time, weekly, monthly — প্রতিটা আলাদা field দিয়ে sort হয়।

```js
// backend/routes/leaderboard.routes.js  (GET /)
const { filter = 'all' } = req.query;
const sortSpec =
    filter === 'weekly'  ? { weeklyPoints: -1 } :
    filter === 'monthly' ? { monthlyPoints: -1 } :
    { points: -1, experience: -1 };   // all-time: points, tie হলে experience

const users = await User.find({})
    .sort(sortSpec).limit(100)
    .select('uid email displayName photoURL points weeklyPoints monthlyPoints experience streak onboarding previousRank createdAt')
    .lean();
```

`.lean()` — plain JS object ফেরত দেয় (Mongoose document না), read-only list-এ দ্রুত ও হালকা।

### ধাপ ৩ — Rank-change snapshot (উঠল না নামল)

"↑↓" badge দেখাতে হলে জানতে হবে আগের বার user কোন rank-এ ছিল। তাই এই request-এই বর্তমান rank গুলো `previousRank`-এ save করা হয় — কিন্তু response ব্লক না করে **fire-and-forget** ভাবে।

```js
// backend/routes/leaderboard.routes.js
(async () => {
    try {
        const bulk = User.collection.initializeUnorderedBulkOp();
        users.forEach((u, idx) => {
            bulk.find({ _id: u._id }).updateOne({ $set: { previousRank: idx + 1 } });
        });
        if (users.length) await bulk.execute();   // bulk write — এক round-trip-এ ১০০ update
    } catch (_) { /* non-critical */ }
})();   // ⬅️ await করা হয়নি — response আটকায় না
```

তারপর response বানানোর সময় rank change হিসাব: `rankChange = previousRank - (idx + 1)` (positive = উঠেছে)।

### ধাপ ৪ — Tier label

Points থেকে একটা tier label বের করা হয় — gamification-এ "আমি Expert" ধরনের অনুভূতি দেয়।

```js
// backend/routes/leaderboard.routes.js
const TIER_LABEL = (pts) => {
    if (pts >= 5000) return { label: 'Legend', icon: '⚡' };
    if (pts >= 2000) return { label: 'Master', icon: '💎' };
    if (pts >= 1000) return { label: 'Expert', icon: '🔥' };
    if (pts >= 500)  return { label: 'Rising', icon: '🌱' };
    return                  { label: 'Rookie', icon: '⭐' };
};
```

### ধাপ ৫ — Weekly/Monthly reset (cron-ready)

Weekly leaderboard অর্থবহ রাখতে প্রতি সোমবার `weeklyPoints` শূন্য করতে হয়। একটা endpoint সব user-এ `updateMany` চালায়।

```js
// backend/routes/leaderboard.routes.js
router.post('/reset-weekly', async (req, res) => {
    await User.updateMany({}, { $set: { weeklyPoints: 0 } });
    res.json({ message: 'Weekly points reset.' });
});
```

(`package.json`-এ `node-cron` আছে — এই endpoint schedule করে চালানোর জন্য।)

## মূল ধারণা (Key Concepts)

- **Gamification loop** — action → reward (XP) → visible rank → motivation → আরও action।
- **Atomic increment (`$inc`)** — concurrent update-এও points নিরাপদে বাড়ে, race ছাড়া।
- **Separate counters per window** — all-time/weekly/monthly আলাদা field, আলাদা reset।
- **Snapshot for delta** — `previousRank` রেখে rank-change (↑↓) দেখানো।
- **Fire-and-forget write** — non-critical update `await` না করে response দ্রুত দেওয়া।
- **Bulk write** — ১০০ update এক round-trip-এ, N+1 এড়ানো।
- **`.lean()`** — read-only query-তে হালকা plain object।
- **Cron-driven reset** — সময়ভিত্তিক maintenance আলাদা scheduled job-এ।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. Points বাড়াতে `$inc` ব্যবহার করলেন কেন, আগে পড়ে তারপর `points = old + 10` করে save না করে?
**উত্তর:** Race condition এড়াতে। "read-modify-write" প্যাটার্নে (আগে points পড়া, তারপর +10 করে save) দুটো action প্রায় একসাথে হলে — যেমন user একটা quiz আর একটা roadmap প্রায় একই সময়ে শেষ করল — দুটোই পুরনো একই মান পড়ে, দুটোই +10 করে save করে, একটা update হারিয়ে যায় (lost update)। `$inc` operation-টা MongoDB সার্ভারে atomic ভাবে করে — "যা আছে তার সাথে যোগ করো" — তাই concurrent increment গুলো সঠিকভাবে জমা হয়, কোনোটা হারায় না। XP-এর মতো accumulating counter-এ `$inc` প্রায় বাধ্যতামূলক।

### ২. all-time/weekly/monthly-র জন্য তিনটা আলাদা field রাখলেন কেন, একটা থেকে হিসাব না করে?
**উত্তর:** কারণ এই তিনটার "reset" আচরণ আলাদা। Weekly points প্রতি সপ্তাহে শূন্য হয়, monthly প্রতি মাসে, all-time কখনো না। যদি শুধু একটা total রাখতাম, তাহলে "এই সপ্তাহে কত পেল" বের করতে প্রতিটা point-earning event timestamp সহ আলাদা log রাখতে হতো এবং প্রতিবার সময়সীমা ধরে যোগ করতে হতো — অনেক ভারী। তিনটা আলাদা counter রাখলে leaderboard শুধু একটা field দিয়ে sort করে, আর reset মানে সেই field-কে শূন্য করা — সহজ ও দ্রুত। Trade-off: এটা denormalized (একই action তিন counter বাড়ায়) এবং reset cron ঠিকমতো না চললে weekly/monthly ভুল দেখাবে; কিন্তু read-heavy leaderboard-এ এই trade-off যুক্তিসঙ্গত।

### ৩. `previousRank` snapshot update-টা `await` করা হয়নি (fire-and-forget) — কেন, এবং এর ঝুঁকি কী?
**উত্তর:** এটা করা হয়েছে response latency কমাতে। Leaderboard fetch-এর মূল কাজ হলো tableটা দ্রুত ফেরত দেওয়া; `previousRank` update করা শুধু পরের বার rank-change দেখানোর জন্য দরকার — user-এর এখনকার response-এর জন্য না। তাই সেই ১০০ user-এর bulk write `await` না করে background-এ ছেড়ে দিয়ে সাথে সাথে response পাঠানো হয়, ফলে user দ্রুত leaderboard পায়। ঝুঁকি: (ক) write ব্যর্থ হলে কেউ জানবে না (silently swallowed, তবে non-critical বলে গ্রহণযোগ্য), (খ) serverless (Vercel) environment-এ response পাঠানোর পর function suspend হয়ে গেলে background write শেষ নাও হতে পারে — এটা এই deployment-এ একটা বাস্তব সমস্যা, কারণ Vercel response-এর পর execution guarantee দেয় না। Production-এ এই ধরনের deferred work আলাদা job/queue-তে করা নিরাপদ।

### ৪. Rank-change হিসাবের এই pattern-এ কী concurrency সমস্যা হতে পারে?
**উত্তর:** `previousRank` প্রতিবার **যে কেউ leaderboard খুললেই** overwrite হয়। মানে দুজন user কাছাকাছি সময়ে leaderboard দেখলে, দ্বিতীয়জনের view প্রথমজনের সদ্য-লেখা `previousRank`-কে "previous" ধরে নেবে — ফলে rank-change প্রায় সবসময় ০ দেখাবে (কারণ snapshot খুব ঘন ঘন নেওয়া হচ্ছে)। অর্থাৎ এই design-এ "previous" মানে "গত কারো-একজনের-view", কোনো নির্দিষ্ট সময়ের snapshot না — যা rank-change-কে অর্থহীন করে দিতে পারে। সঠিক approach হতো snapshot নেওয়া নির্দিষ্ট বিরতিতে (যেমন দৈনিক cron), user-এর প্রতিটা view-তে না — তাহলে "গতকালের rank বনাম আজকের" একটা meaningful delta হতো।

### ৫. `.lean()` ব্যবহার করলেন কেন এই query-তে?
**উত্তর:** Default-এ Mongoose প্রতিটা document-কে একটা full Mongoose document object-এ পরিণত করে — যাতে getter/setter, virtual, `save()`, change tracking সব থাকে; এটা memory ও CPU খরচ করে। কিন্তু leaderboard-এ আমরা শুধু data পড়ে JSON হিসেবে পাঠাচ্ছি, কোনো document method বা save লাগছে না। `.lean()` দিলে Mongoose সরাসরি plain JavaScript object ফেরত দেয় — অনেক হালকা ও দ্রুত, বিশেষত ১০০ document-এ। যেখানে শুধু read করে serialize করব, সেখানে `.lean()` একটা সহজ ও কার্যকর optimization। ব্যতিক্রম — যদি document mutate করে save করতে হতো, তখন lean ব্যবহার করা যেত না।

### ৬. Reset-weekly endpoint কি নিরাপদ? যে কেউ কল করলে কী হবে?
**উত্তর:** এখন এটা মোটেও সুরক্ষিত না — একটা open `POST /api/leaderboard/reset-weekly`, কোনো authentication/authorization নেই। যে কেউ এই URL-এ POST পাঠালে সব user-এর `weeklyPoints` শূন্য হয়ে যাবে — এটা একটা destructive, sabotage-প্রবণ endpoint। এটা শুধুই cron/internal থেকে চালানোর কথা, কিন্তু কোড সেটা enforce করে না। সুরক্ষিত করতে: (ক) একটা secret token/header দিয়ে verify করা (`if (req.headers['x-cron-secret'] !== process.env.CRON_SECRET) return 403`), (খ) বা এটাকে HTTP endpoint না বানিয়ে সরাসরি একটা scheduled job function হিসেবে চালানো যা বাইরে থেকে reachable না। এটা এই সিস্টেমের একটা গুরুত্বপূর্ণ নিরাপত্তা ঘাটতি।

### ৭. `.limit(100)` করে top-100 আনছেন — কোনো user 101তম হলে সে নিজের rank কীভাবে দেখবে?
**উত্তর:** এই top-100 list-এ সে থাকবে না, তাই leaderboard table-এ নিজেকে খুঁজে পাবে না — একটা সীমাবদ্ধতা। তবে কোডে আলাদা একটা endpoint আছে — `GET /api/leaderboard/user-points?email=...` — যা `countDocuments({ points: { $gt: basePoints } }) + 1` দিয়ে ঐ নির্দিষ্ট user-এর সঠিক rank হিসাব করে, top-100-এ না থাকলেও। অর্থাৎ list দেখায় top-100, কিন্তু "your rank" আলাদাভাবে গোটা collection-এর বিপরীতে গণনা করা হয়। এটা scale-এর জন্য যুক্তিসঙ্গত — পুরো লক্ষ user render করা অর্থহীন, কিন্তু প্রত্যেকে নিজের অবস্থান জানতে পারে। Count query বড় হলে ধীর হতে পারে, তখন index (`points` descending) ও caching লাগবে।

### ৮. Points-এর producer অনেক জায়গায় (quiz, roadmap, streak) ছড়ানো — এটা maintainable? 
**উত্তর:** এখন এটা duplication-এ ভুগছে। quiz controller, roadmap controller, leaderboard `add-points` — তিন জায়গায় `$inc points` এবং streak update-এর প্রায় একই কোড। সমস্যা: point value বা streak নিয়ম বদলাতে হলে সব জায়গা খুঁজে বদলাতে হয়, একটা ভুলে গেলে inconsistent XP economy হয়। আরও, quiz `points` বাড়ায় কিন্তু `weeklyPoints`/`monthlyPoints` বাড়ায় না (roadmap বাড়ায়) — এই অসঙ্গতি weekly leaderboard-কে ভুল করবে। Maintainable করতে একটা কেন্দ্রীয় `awardPoints(user, amount, reason)` service বানাতাম যা points/weekly/monthly/experience সব একসাথে বাড়ায়, streak update করে, notification দেয় — সব producer শুধু সেটা কল করত। এটা DRY ও single source of truth দিত।

### ৯. Leaderboard-এ real-time update চাইলে (কেউ points পেলে সাথে সাথে rank বদলাক) কীভাবে করতেন?
**উত্তর:** এখন leaderboard pull-based — user page খুললে তখনকার snapshot দেখে। Real-time করতে WebSocket (যেমন Socket.IO) বা Server-Sent Events ব্যবহার করতাম: কেউ points পেলে সার্ভার connected client-দের একটা event push করত, frontend সেই অনুযায়ী list re-render করত। তবে সত্যিকারের live leaderboard ব্যয়বহুল ও প্রায়ই অপ্রয়োজনীয় — প্রতিটা point change-এ ১০০ জনের rank recompute ও broadcast করা ভারী। বাস্তবে একটা middle ground ভালো: leaderboard একটা cache (যেমন Redis sorted set) এ রাখা যেখানে `ZINCRBY`/`ZRANK` দিয়ে rank O(log n)-এ পাওয়া যায়, আর client কয়েক সেকেন্ড অন্তর poll করে বা push পায়। MongoDB sort-scan-এর চেয়ে Redis sorted set leaderboard-এর জন্য purpose-built।

### ১০. Tier threshold (500/1000/2000/5000) hardcoded — এই gamification design-এ কী দুর্বলতা দেখছেন, কীভাবে ভালো করতেন?
**উত্তর:** কয়েকটা দুর্বলতা: (ক) threshold গুলো magic number, code-এ hardcoded — balance করতে (যেমন "Expert" পাওয়া খুব সহজ/কঠিন) code deploy করতে হয়, config/DB-তে থাকলে tune করা সহজ হতো। (খ) Tier শুধু all-time points-এর উপর, তাই একবার Legend হলে চিরকাল Legend — নিষ্ক্রিয় হলেও; recency (weekly activity) মেশালে tier আরও অর্থবহ হতো। (গ) Points সহজে "farm" করা যায় — শুধু login/roadmap বানিয়ে (কোনো বাস্তব শেখা ছাড়াই) points বাড়ানো যায়, যা leaderboard-কে vanity বানায়। ভালো design: points-কে verified learning outcome (quiz score, milestone সম্পন্ন)-এর সাথে শক্তভাবে বাঁধা, anti-abuse (rate limit, diminishing returns) রাখা, এবং tier/threshold config-driven করা যাতে data দেখে balance করা যায়।
