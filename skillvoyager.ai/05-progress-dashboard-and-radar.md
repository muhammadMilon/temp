# 05 — Progress Tracking Dashboard & Skill Strength Radar

## এই ফিচারটা আসলে কী?

একজন learner-এর জন্য সবচেয়ে বড় motivation হলো নিজের অগ্রগতি দেখতে পারা। SkillVoyager-এর **Progress Dashboard** (`/progress`) সেটাই দেখায় — কত শতাংশ শেষ, কোন milestone-এ আছে, কত দিন বাকি, আর সবচেয়ে আকর্ষণীয় অংশ **Skill Strength Radar** — একটা মাকড়সার জালের মতো chart যেখানে প্রতিটা skill-এ user কতটা শক্তিশালী তা visually দেখা যায়।

Backend-এ এই data রাখে আলাদা একটা `Progress` collection। এর সবচেয়ে interesting field হলো `skillStrength` — এটা একটা **Mongoose `Map`** (key = skill নাম, value = 0–100 শক্তি)। এই design বেছে নেওয়ার কারণ — user-ভেদে skill-এর সংখ্যা ও নাম আলাদা, তাই fixed column না রেখে dynamic key-value দরকার।

মজার ব্যাপার — এই skill value গুলো নিজে থেকেই বাড়ে: user quiz দিলে, বা milestone শেষ করলে সংশ্লিষ্ট skill-এ boost পড়ে (দেখুন [09-ai-quiz](09-ai-quiz-generation-and-evaluation.md))। অর্থাৎ dashboard একটা static report না, এটা user-এর activity-র live প্রতিফলন।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| DB Model | `backend/models/Progress.js` | `Map` skillStrength, milestones, watchHistory |
| API | `backend/routes/progress.routes.js` | fetch/auto-create, milestone update, save-roadmap |
| Controller | `backend/controllers/progress.controller.js` | `getProgress` (default data সহ create) |
| UI | `frontend/src/pages/Dashboard/ProgressDashboard.jsx` | dashboard fetch করে render করে |
| UI (chart) | `frontend/src/components/SkillRadarChart.jsx` | Chart.js radar render করে |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`Progress.js`** — schema যেখানে `skillStrength` একটা `Map of Number`, সাথে `milestones` array আর daily `watchHistory`; user-ভিত্তিক (`uid` unique)।
- **`progress.routes.js`** — data না থাকলে default বানায়, milestone complete করলে percentage auto-calculate ও skill boost করে, roadmap-কে milestone-এ রূপান্তর করে।
- **`ProgressDashboard.jsx`** — `useEffect`-এ `/api/progress?uid=...` থেকে data এনে stat card, milestone timeline ও radar render করে।
- **`SkillRadarChart.jsx`** — chart.js-এর radar chart; দুটো dataset — "Your Skill Strength" ও একটা flat "Target Level" (৮০%)।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — `Map` দিয়ে dynamic skill storage

`Progress.js`-এ `skillStrength`-কে `Map` করা হয়েছে, fixed field না রেখে। এতে যেকোনো user-এর যেকোনো skill (react, docker, python...) key হিসেবে বসতে পারে।

```js
// backend/models/Progress.js
const progressSchema = new mongoose.Schema({
    uid: { type: String, required: true, unique: true },
    percentage: { type: Number, default: 0 },
    targetRole: { type: String, default: "Full Stack Developer" },
    skillStrength: { type: Map, of: Number, default: () => new Map() },   // ⬅️ dynamic key-value
    milestones: { type: [milestoneSchema], default: [] },
    watchHistory: [{ date: String, hours: Number }],                      // activity chart
}, { timestamps: true });
```

### ধাপ ২ — Data না থাকলে auto-create (lazy initialization)

প্রথমবার dashboard খুললে হয়তো progress record নেই। Endpoint তখন একটা sensible default বানিয়ে দেয় — user খালি screen দেখে না।

```js
// backend/routes/progress.routes.js
let progress = await Progress.findOne({ uid: String(uid) });
if (!progress) {
    progress = new Progress({
        uid: String(uid), percentage: 0,
        skillStrength: new Map([["React",40],["JavaScript",40],["Node.js",40],["Git",40],["UI/UX",40],["Python",40]]),
        milestones: [
            { title: "Onboarding Complete", status: "completed", date: new Date().toISOString().split('T')[0] },
            { title: "First Quiz", status: "upcoming", date: "TBD" }
        ],
        estimatedDays: 90,
    });
    await progress.save();
}
res.status(200).json({ success: true, data: progress });
```

### ধাপ ৩ — Milestone complete করলে percentage auto-calculate

এটা business logic-এর সুন্দর অংশ। কোনো milestone "completed" হলে সিস্টেম নিজেই শতাংশ হিসাব করে, সংশ্লিষ্ট skill boost করে, notification বানায়, আর পরের milestone-কে "current" করে।

```js
// backend/routes/progress.routes.js  (PATCH /milestone/:uid)
const total = progress.milestones.length;
const completed = progress.milestones.filter(m => m.status === 'completed').length;
if (total > 0) progress.percentage = Math.round((completed / total) * 100);   // ⬅️ auto %

// milestone title-এ keyword থাকলে সেই skill +10 boost
const keywords = ['react','node','javascript','css','html','python','mongodb','docker','typescript'];
keywords.forEach(key => {
    if (milestoneTitle.toLowerCase().includes(key)) {
        const cur = progress.skillStrength.get(key) || 0;
        progress.skillStrength.set(key, Math.min(100, cur + 10));   // ⬅️ cap at 100
    }
});
```

`Math.min(100, ...)` — skill কখনো ১০০-র বেশি যায় না; `progress.skillStrength.get/set` — Map API ব্যবহার।

### ধাপ ৪ — Frontend data fetch

`ProgressDashboard` mount হলে uid দিয়ে data আনে; না থাকলে `demo-user` fallback।

```jsx
// frontend/src/pages/Dashboard/ProgressDashboard.jsx
useEffect(() => {
    (async () => {
        const uid = user?.uid || 'demo-user';
        const r = await fetch(`${API_BASE}/api/progress?uid=${uid}`);
        const d = await r.json();
        if (d.success) setProgressData(d.data);
    })();
}, [user]);
```

### ধাপ ৫ — Radar chart render

`SkillRadarChart` chart.js-এর `Radar` ব্যবহার করে। মূল কৌশল — user-এর skill-কে একটা polygon, আর একটা flat "Target 80%" dashed polygon পাশে দেখানো, যাতে user gap বুঝতে পারে।

```jsx
// frontend/src/components/SkillRadarChart.jsx
const data = {
    labels: Object.keys(skillsData || {}),                 // skill নাম
    datasets: [
        { label: 'Your Skill Strength', data: Object.values(skillsData || {}), /* teal fill */ },
        { label: 'Target Level',
          data: Array(Object.keys(skillsData || {}).length).fill(80),   // ⬅️ সব axis-এ ৮০
          borderDash: [5, 5], pointRadius: 0 },
    ],
};
const options = { scales: { r: { min: 0, max: 100 } } };   // radar range ঠিক করা
```

গুরুত্বপূর্ণ: chart.js-এর component গুলো আগে `ChartJS.register(...)` করতে হয় (RadialLinearScale, PointElement...), নাহলে radar render হবে না।

## মূল ধারণা (Key Concepts)

- **Mongoose `Map` type** — dynamic key-value (schema-তে column না জেনেও) সংরক্ষণ।
- **Lazy initialization** — data প্রথমবার চাইলে তখনই default বানানো, আগে থেকে না।
- **Derived state** — percentage সংরক্ষণ না করে completed/total থেকে হিসাব করে বসানো।
- **Event-driven skill boost** — milestone/quiz-এর মতো action skill value বাড়ায় (side effect)।
- **Data visualization** — কাঁচা number-কে radar polygon-এ রূপান্তর করে insight দেওয়া।
- **Registration pattern (chart.js)** — শুধু দরকারি chart module register করে bundle ছোট রাখা।
- **Target overlay** — actual-এর পাশে target দেখিয়ে gap বোঝানো।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. `skillStrength`-কে `Map` করলেন কেন, আলাদা field/subdocument না বানিয়ে?
**উত্তর:** কারণ skill set dynamic — একেক user-এর একেক রকম ও ভিন্ন সংখ্যক skill (কারো ৩টা, কারো ১০টা; নাম-ও আলাদা)। যদি fixed field (`react`, `node`...) রাখতাম, নতুন skill যোগ করতে schema বদলাতে হতো এবং যে user-এর সেই skill নেই তার doc-এ অপ্রয়োজনীয় null field থাকত। `Map of Number` দিলে যেকোনো skill নাম key হিসেবে বসে, value 0–100 — schema না বদলে infinite skill support করা যায়, আর "get/set/has"-এর মতো Map API পাওয়া যায়। Trade-off: Map-এর key-তে query করা array/subdocument-এর চেয়ে একটু কম নমনীয় (indexing সীমিত), কিন্তু এখানে আমরা পুরো Map একসাথে পড়ি, তাই সেটা সমস্যা না।

### ২. percentage DB-তে save করছেন আবার milestone থেকে হিসাবও করছেন — এটা কি contradiction? কোনটা সত্য?
**উত্তর:** এখানে percentage হলো **derived-then-cached** — milestone update-এর সময় `completed/total` থেকে হিসাব করে তারপর `progress.percentage`-এ save করা হয়। এটা করার কারণ read fast রাখা (frontend প্রতিবার milestone গুণে বের না করে সরাসরি percentage পড়ে)। ঝুঁকি হলো — যদি কেউ সরাসরি milestone array বদলায় কিন্তু এই PATCH route দিয়ে না, তখন saved percentage আর আসল অবস্থা mismatch হবে (stale data)। বিশুদ্ধ approach হতো percentage কখনো save না করে সবসময় on-the-fly হিসাব করা (single source of truth = milestones)। এখানে performance-এর জন্য cache করা হয়েছে, কিন্তু consistency নিশ্চিত করতে সব milestone-mutation একই path দিয়ে যেতে হবে।

### ৩. Milestone title-এ keyword খুঁজে skill boost করা — এই approach-এর দুর্বলতা কী?
**উত্তর:** এটা fragile string-matching। "React Basics" milestone শেষ করলে `react` skill +10 পায়, কিন্তু নির্ভর করছে title-এ ঠিক সেই keyword থাকার উপর। "Frontend Fundamentals" বা "Component Architecture" milestone React শেখালেও keyword না মেলায় কোনো boost পাবে না — false negative। উল্টো, "React vs Angular comparison" title-এ react থাকলে boost পাবে যদিও হয়তো skill বাড়েনি — false positive। এটা brittle কারণ semantic না, শুধু substring। উন্নত করতে milestone-এ explicit `skillsAffected: ['react']` field রাখতাম (title-এর উপর নির্ভর না করে), যাতে boost deterministic ও নির্ভুল হয়।

### ৪. Lazy initialization (data চাইলে তখন default বানানো) বনাম onboarding-এ আগেই বানানো — কোনটা ভালো?
**উত্তর:** দুটোরই যুক্তি আছে, এবং এখানে আসলে **দুটোই ঘটছে** — onboarding সম্পন্ন করলে Progress বানানো হয়, আবার `/api/progress` fetch করলে না থাকলেও বানানো হয়। Lazy init-এর সুবিধা — user onboarding skip করলেও (বা data কোনোভাবে হারালে) dashboard খালি না দেখিয়ে default দেয়, robustness বাড়ে। অসুবিধা — read endpoint-এ write side-effect (GET request DB-তে লিখছে) idempotency ভাঙে এবং RESTful না। দুই জায়গায় default থাকায় দুটো আলাদা default set-ও আছে (একটু ভিন্ন), যা inconsistency-র উৎস। আদর্শ হতো — এক জায়গায় (onboarding বা একটা shared `ensureProgress` helper) default সংজ্ঞায়িত করা, GET-কে pure রাখা।

### ৫. Radar chart-এ "Target Level 80%" flat overlay রাখলেন কেন?
**উত্তর:** এটা একটা reference/benchmark দেয়। শুধু user-এর skill polygon দেখালে user বুঝত না "কতটা যথেষ্ট"। ৮০%-এর একটা dashed target polygon পাশে থাকলে, user সহজেই দেখতে পারে কোন skill target ছুঁয়েছে আর কোথায় gap বড় — অর্থাৎ কোনটায় বেশি মনোযোগ দিতে হবে। Visualization-এ actual বনাম goal পাশাপাশি দেখানো একটা শক্তিশালী কৌশল, কারণ মানুষ relative distance দ্রুত বোঝে। দুর্বলতা — ৮০% hardcoded ও সব skill-এ একই; বাস্তবে target role-ভেদে skill-এর গুরুত্ব আলাদা, তাই per-skill weighted target আরও অর্থবহ হতো।

### ৬. chart.js-এ `ChartJS.register(...)` না করলে কী হয়, এটা কেন দরকার?
**উত্তর:** chart.js v3+ **tree-shakable** — অর্থাৎ পুরো library import না করে শুধু যে অংশ দরকার (RadialLinearScale, PointElement, LineElement, Filler, Tooltip, Legend) সেগুলো explicitly register করতে হয়। Register না করলে radar-এর প্রয়োজনীয় scale/element পাওয়া যাবে না এবং chart render-এ error বা blank দেখাবে। এই design-এর সুবিধা — অ্যাপ শুধু ব্যবহৃত chart feature গুলো bundle-এ নেয়, ফলে bundle ছোট থাকে (bar/pie/line-এর অব্যবহৃত কোড বাদ)। এটা "explicit over implicit" — একটু বেশি setup, কিন্তু performance ভালো।

### ৭. `Math.min(100, cur + 10)` — এই cap না থাকলে কী সমস্যা হতো?
**উত্তর:** Skill value 100-এর বেশি হয়ে যেত। Radar chart-এর scale `min:0, max:100`, তাই ১০০-র বেশি value চার্টের বাইরে চলে যেত বা clip হতো — visual ভুল দেখাত। আরও গুরুত্বপূর্ণ, semantically 0–100 একটা percentage/mastery scale; ১২০% "mastery" অর্থহীন। Cap দিয়ে value সবসময় valid range-এ রাখা হয়, ফলে chart, comparison, target-gap সব সঠিক থাকে। এটা একটা ছোট কিন্তু জরুরি invariant — যেকোনো bounded metric-এ boost করার সময় ceiling রাখা উচিত।

### ৮. Dashboard-এ data fetch `useEffect`-এ হচ্ছে — এর সীমাবদ্ধতা কী, কীভাবে উন্নত করতেন?
**উত্তর:** `useEffect`-এ manual fetch-এ কয়েকটা সীমাবদ্ধতা: (ক) কোনো caching নেই — প্রতিবার page-এ এলে নতুন request, (খ) loading/error state নিজে manage করতে হয়, (গ) কোনো background refetch/stale-while-revalidate নেই, (ঘ) দুই component একই data চাইলে দুবার fetch হয়। এছাড়া একটা bug-ও আছে — GET request data না থাকলে DB-তে write করে, যা fetch-কে side-effect-ful করে। উন্নত করতে **React Query (TanStack Query)** বা SWR ব্যবহার করতাম — এরা caching, dedup, retry, background refetch সব দেয়, code কমে আর UX ভালো হয়। বড় হলে এটা প্রায় অবশ্যম্ভাবী।

### ৯. এই progress সিস্টেম কীভাবে অন্য feature-এর সাথে coupled? সেটা ভালো না খারাপ?
**উত্তর:** `Progress` অনেক feature-এর সাথে যুক্ত: onboarding এটা initialize করে, quiz এতে skillStrength boost করে (topic থেকে key বানিয়ে), roadmap generator save-roadmap দিয়ে milestone বসায়, dashboard/radar এটা পড়ে। এই coupling ইতিবাচক দিক থেকে — এটা একটা central "learning state" যা পুরো অ্যাপকে একসূত্রে বাঁধে, quiz দিলে সত্যিই radar-এ প্রতিফলন দেখা যায় (satisfying UX)। নেতিবাচক দিক — অনেক জায়গা থেকে এই collection লেখে, তাই skill-key normalization একরকম রাখা জরুরি (নাহলে quiz-এর `react` আর onboarding-এর key না মিললে boost হারিয়ে যাবে), এবং একটা জায়গার bug অন্য জায়গায় প্রকাশ পায়। এই ঝুঁকি কমাতে skill-key normalization ও Progress mutation গুলোকে একটা shared service-এ কেন্দ্রীভূত করা উচিত।

### ১০. `watchHistory`-তে date কেন `String` ("2026-03-25") রাখা হলো `Date` না রেখে? Trade-off?
**উত্তর:** `watchHistory`-তে date `"YYYY-MM-DD"` string হিসেবে রাখা হয়েছে যাতে সেটা সরাসরি একটা দিনের bucket key হিসেবে কাজ করে — activity chart-এ "কোন দিনে কত ঘণ্টা" গোছাতে সুবিধা, timezone/time-of-day নিয়ে ভাবতে হয় না, আর frontend-এ সরাসরি label হিসেবে দেখানো যায়। Trade-off: string date-এ range query (যেমন "শেষ ৭ দিন") বা date arithmetic করা `Date` type-এর চেয়ে কঠিন ও কম নির্ভরযোগ্য (lexicographic sort কাজ করে কারণ ISO format, কিন্তু month/week rollup করতে parse করতে হয়)। যদি heavy date-based analytics লাগত, তখন `Date` type + proper index ভালো হতো; এখানে simple daily-bucket use case-এ string যথেষ্ট এবং সহজ।
