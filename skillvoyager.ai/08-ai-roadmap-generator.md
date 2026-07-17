# 08 — AI Roadmap Generator (TypeScript + AI + Notifications)

## এই ফিচারটা আসলে কী?

এটাই SkillVoyager-এর নামের পেছনের মূল feature — user একটা **career goal** (যেমন "Full Stack Developer"), একটা **timeline** আর **learning style** দেয়, আর AI তার জন্য একটা ধাপে-ধাপে **personalized roadmap** বানায়: catchy title, description, আর ৩–৫টা phase (প্রতিটায় duration, topics, recommended resources)।

Backend `callGemini` দিয়ে Gemini-কে একটা কড়া নির্দেশ দেয় — "শুধু এই JSON structure-এ উত্তর দাও"। AI যে text ফেরত দেয় তা থেকে JSON অংশটা বের করে parse করা হয়, DB-তে save করা হয়, আর side-effect হিসেবে user-কে **+৫০ XP** ও একটা **notification** দেওয়া হয়।

Frontend দিকটা লেখা হয়েছে **TypeScript**-এ (`RoadmapGenerator.tsx`) — প্রজেক্টের বেশিরভাগ JSX হলেও এই feature-টা typed। এতে form → API → animated result rendering পুরোটা আছে, সাথে একটা client-side retry (AI busy হলে) আর "Save Roadmap" (roadmap-কে progress milestone-এ রূপান্তর)।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| UI | `frontend/src/pages/Roadmaps/RoadmapGenerator.tsx` | form, API call, client retry, result render |
| API | `backend/routes/roadmap.routes.js` | `POST /api/roadmap/generate` route |
| Controller | `backend/controllers/roadmap.controller.js` | prompt → Gemini → JSON parse → save → XP + notif |
| Core Service | `backend/services/gemini.service.js` | আসল Gemini call ([দেখুন 07](07-gemini-ai-integration-layer.md)) |
| DB Model | `backend/models/Roadmap.js`, `backend/models/Notification.js` | roadmap ও notification সংরক্ষণ |
| Persist | `backend/routes/progress.routes.js` (`/save-roadmap`) | roadmap → progress milestones |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`RoadmapGenerator.tsx`** — typed frontend; form state, fetch, "AI busy" হলে ৩ বার client-side retry, আর framer-motion দিয়ে animated phase card।
- **`roadmap.controller.js`** — feature-এর brain: structured prompt বানায়, Gemini-এর text থেকে JSON extract করে, `Roadmap` save করে, XP ও Notification যোগ করে।
- **`Roadmap.js` / `Notification.js`** — roadmap (phase array সহ) ও notification-এর schema।
- **`progress.routes.js` → `/save-roadmap`** — generated roadmap-কে user-এর progress-এর milestone list-এ রূপান্তর করে।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — Structured prompt (AI-কে JSON-এ বাধ্য করা)

Controller একটা বিস্তারিত prompt বানায় যা AI-কে ঠিক কোন কাঠামোতে উত্তর দিতে হবে বলে দেয়। "Return ONLY a JSON object" লাইনটা গুরুত্বপূর্ণ।

```js
// backend/controllers/roadmap.controller.js
const prompt = `Create a detailed learning roadmap for:
Goal: "${goal}"
Timeline: "${timeline}"
Learning Style: "${learningStyle}"
...
Return ONLY a JSON object:
{
  "title": "...", "description": "...",
  "phases": [ { "phaseName": "...", "duration": "...", "description": "...",
               "topics": ["..."], "resources": ["..."] } ]
}`;
const responseText = await callGemini(prompt);
```

### ধাপ ২ — JSON extract (LLM output সাফ করা)

LLM প্রায়ই JSON-এর আগে/পরে কিছু text জুড়ে দেয় ("Here is your roadmap:" ইত্যাদি)। তাই regex দিয়ে প্রথম `{` থেকে শেষ `}` পর্যন্ত অংশটা টেনে বের করে parse করা হয়।

```js
// backend/controllers/roadmap.controller.js
const jsonMatch = responseText.match(/\{[\s\S]*\}/);   // ⬅️ প্রথম { থেকে শেষ } পর্যন্ত
if (!jsonMatch) throw new Error("Invalid AI response format");
const roadmapData = JSON.parse(jsonMatch[0]);
```

### ধাপ ৩ — DB-তে save

Parse হওয়া data + user input মিলিয়ে একটা `Roadmap` document বানিয়ে save।

```js
// backend/controllers/roadmap.controller.js
const newRoadmap = new Roadmap({
    userEmail: userEmail || "anonymous",
    goal, timeline, learningStyle,
    ...roadmapData          // title, description, phases
});
await newRoadmap.save();
```

### ধাপ ৪ — Side-effects: XP + Notification

Roadmap বানানো একটা meaningful action, তাই user +৫০ XP পায় (streak update সহ) এবং একটা notification তৈরি হয়। খেয়াল করুন — `anonymous` user হলে এসব skip হয়।

```js
// backend/controllers/roadmap.controller.js
if (userEmail && userEmail !== "anonymous") {
    await User.findOneAndUpdate({ email: userEmail }, {
        $inc: { points: 50, weeklyPoints: 50, monthlyPoints: 50, experience: 50 },
        $set: { streak: newStreak, lastStreakDate: today }
    });
    const newNotif = new Notification({
        userId: u.uid, type: 'roadmap_update',
        title: 'New Roadmap Generated',
        message: `You successfully generated a new roadmap: ${roadmapData.title}`,
        link: '/roadmap/generate'
    });
    await newNotif.save();
}
```

### ধাপ ৫ — Frontend: client-side retry (AI busy হলে)

Backend-এ retry থাকা সত্ত্বেও frontend-এও একটা recursive retry আছে — "busy"/"quota" error এলে user-কে জানিয়ে অপেক্ষা করে আবার চেষ্টা।

```tsx
// frontend/src/pages/Roadmaps/RoadmapGenerator.tsx
const tryGenerate = async (attempt) => {
    const response = await fetch(`${API_BASE}/api/roadmap/generate`, { method:'POST', /* body */ });
    const data = await response.json();
    if (!response.ok) {
        const isRateLimit = data.error?.toLowerCase().includes('busy') || data.error?.toLowerCase().includes('quota');
        if (isRateLimit && attempt < 3) {
            const waitSec = attempt * 6;
            toast.info(`AI is busy, retrying in ${waitSec}s… (attempt ${attempt}/3)`);
            await delay(waitSec * 1000);
            return tryGenerate(attempt + 1);   // ⬅️ recursive retry
        }
        throw new Error(data.error || 'Failed to generate roadmap');
    }
    return data;
};
```

### ধাপ ৬ — "Save Roadmap" → progress milestones

User চাইলে generated roadmap-কে তার progress-এ save করতে পারে — তখন phase গুলো milestone হয়ে যায় (প্রথমটা "current", বাকি "upcoming")।

```js
// backend/routes/progress.routes.js  (POST /save-roadmap)
const newMilestones = roadmapData.phases.map((phase, idx) => ({
    title: phase.phaseName,
    status: idx === 0 ? 'current' : 'upcoming',   // প্রথম phase active
    date: 'TBD'
}));
progress.milestones = newMilestones;
progress.percentage = 0;                          // নতুন roadmap → progress reset
progress.targetRole = roadmapData.title || progress.targetRole;
await progress.save();
```

## মূল ধারণা (Key Concepts)

- **Prompt engineering** — AI-কে exact output structure (JSON schema) নির্দেশ দিয়ে predictable বানানো।
- **Robust parsing** — LLM-এর "নোংরা" text থেকে regex দিয়ে JSON বের করা।
- **Side-effects on success** — মূল কাজ (roadmap) + সংশ্লিষ্ট update (XP, notification) একসাথে।
- **Defense in depth (retry)** — backend ও frontend দুই স্তরেই retry — resilience।
- **TypeScript on the frontend** — typed props/state দিয়ে UI-তে ভুল কমানো।
- **Feature integration** — roadmap → progress → dashboard/radar, সব একসূত্রে।
- **Graceful anonymous handling** — logged-out user-ও roadmap পায়, শুধু XP/notif skip।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. AI-কে JSON-এ উত্তর দিতে বলছেন — কিন্তু LLM তো non-deterministic; ভুল/অসম্পূর্ণ JSON দিলে কী হয়?
**উত্তর:** এটা এই approach-এর মূল ঝুঁকি। LLM কখনো JSON-এর বাইরে text জুড়ে দিতে পারে, বা trailing comma/অসম্পূর্ণ bracket দিতে পারে — তখন `JSON.parse` throw করবে। এখানে দুই স্তরে সুরক্ষা: (ক) regex `/\{[\s\S]*\}/` দিয়ে শুধু `{...}` অংশ টেনে নিয়ে চারপাশের text বাদ দেওয়া, (খ) পুরোটা `try/catch`-এ — parse fail করলে roadmap-এ 500 error দিয়ে user retry করে, আর skill-gap controller-এ তো fallback হিসেবে raw text-টাই guide হিসেবে দেখানো হয়। তবে এটা নিখুঁত না — regex greedy, তাই text-এ দুটো `{...}` block থাকলে ভুল ধরতে পারে। সবচেয়ে robust হতো Gemini-এর **structured output / responseSchema** feature ব্যবহার করা, যা model-কে valid JSON দিতে বাধ্য করে — তখন regex hack লাগত না।

### ২. Backend-এ retry আছে ([07 দেখুন](07-gemini-ai-integration-layer.md)), তবু frontend-এও retry রাখলেন কেন? এটা কি redundant?
**উত্তর:** সম্পূর্ণ redundant না, দুটো ভিন্ন স্তরের ব্যর্থতা ধরে। Backend retry ধরে Gemini API-এর transient 503/429 — কিন্তু সেটাও ব্যর্থ হয়ে user-এর কাছে error গেলে, frontend retry ধরে সেই final failure-টাকে (এবং network glitch, cold start ইত্যাদি) এবং user-কে বেশি অপেক্ষা করায় (৬s, ১২s) আরেকবার সুযোগ দেয়। এটা "defense in depth" — এক স্তর ফেল করলে পরের স্তর ধরে। তবে দুর্বলতা: এতে worst case-এ user অনেকক্ষণ (backend ~১২s × frontend ৩ চেষ্টা) আটকে থাকতে পারে, এবং প্রতিটা frontend retry নতুন করে Gemini call করে খরচ বাড়ায়। ভালো হতো — একটা জায়গায় (backend) resilient retry রেখে frontend-এ শুধু single graceful error + manual "Try again" বাটন দেওয়া।

### ৩. `anonymous` user-এর জন্য XP/notification skip করলেন কেন? এটা কি ইচ্ছাকৃত?
**উত্তর:** হ্যাঁ, ইচ্ছাকৃত এবং সঠিক। XP, streak, notification সব একটা নির্দিষ্ট user account-এর সাথে যুক্ত (email/uid দিয়ে খুঁজে update)। `anonymous` মানে logged-out — এর কোনো account নেই, তাই কার points বাড়াবে বা কাকে notification দেবে সেটা নির্ধারণ করা যায় না। তাই `if (userEmail && userEmail !== "anonymous")` দিয়ে এই side-effect গুলো শুধু real user-এর জন্য চলে। এতে logged-out user-ও roadmap generate করে দেখতে পারে (product try করার সুযোগ, conversion বাড়ায়), শুধু persistence/reward পায় না। এটা একটা ভালো UX + data-integrity ভারসাম্য।

### ৪. এই controller-এ অনেকগুলো await পরপর (Gemini, save, User update, Notification save) — একটা fail করলে কী হয়? Transaction দরকার ছিল?
**উত্তর:** এখন এগুলো একটার পর একটা চলে, atomic না। যদি roadmap save হয়ে যায় কিন্তু তারপর User XP update fail করে, তাহলে roadmap থাকবে কিন্তু XP পাবে না — partial state। তবে খেয়াল করলে দেখা যায় XP/notification অংশটা আলাদা `try/catch`-এ মোড়া, যাতে সেটা fail করলেও roadmap response ঠিক যায় ("XP error" শুধু log হয়) — অর্থাৎ core কাজ (roadmap) বাঁচানো হয়েছে, side-effect fail হলেও user roadmap পায়। এটা pragmatic, কারণ XP হারানো roadmap হারানোর চেয়ে কম গুরুতর। যদি এই operation গুলো সবই "সব হবে নাহলে কিছুই না" হতো (যেমন payment), তখন MongoDB transaction দরকার হতো। এখানে eventual/best-effort consistency গ্রহণযোগ্য, তাই full transaction over-engineering হতো।

### ৫. Frontend TypeScript, বাকি প্রজেক্ট বেশিরভাগ JSX — এই mixing কি সমস্যা? কেন এটা করলেন?
**উত্তর:** Vite/React TS ও JS ফাইল একসাথে compile করতে পারে, তাই technically সমস্যা হয় না — একটা `.tsx` অন্য `.jsx` import করতে পারে। এই feature-টা TS-এ করার সম্ভাব্য কারণ — roadmap-এর data structure (phases, topics, resources) nested ও জটিল, তাই typed interface থাকলে render করার সময় ভুল (যেমন `phase.topic` বনাম `phase.topics`) compile-time-এ ধরা পড়ে। দুর্বলতা: mixed codebase-এ consistency কমে — কিছু ফাইল type-safe, কিছু না; নতুন dev বিভ্রান্ত হতে পারে; আর যেহেতু বাকি অংশ untyped, এই ফাইলের type-safety-র পূর্ণ সুবিধা (end-to-end typing) পাওয়া যায় না (API response এখনো `any`)। আদর্শ হতো পুরো frontend ধীরে ধীরে TS-এ migrate করা, নাহলে অন্তত shared type রাখা।

### ৬. "Save Roadmap" আলাদা একটা action — generate করার সময়ই save না করে কেন?
**উত্তর:** দুটো আলাদা রাখা UX ও data-এর দিক থেকে অর্থবহ। Generate করলে roadmap `Roadmap` collection-এ history হিসেবে সবসময় save হয় (যা তৈরি হলো তার record), কিন্তু user সেটা পছন্দ নাও করতে পারে — হয়তো আরেকবার ভিন্ন goal দিয়ে generate করবে। "Save Roadmap" আলাদা রাখায় user preview দেখে **সিদ্ধান্ত** নেয় এটাই তার active learning path কিনা; save করলে তবেই এটা তার `Progress`-এর milestone হয়ে dashboard-এ আসে এবং percentage reset হয়। এটা "browse then commit" pattern — কোনো generate করা roadmap স্বয়ংক্রিয়ভাবে তার active plan হয়ে অন্য progress মুছে দেয় না। ব্যবহারকারীকে control দেওয়া।

### ৭. Regex `/\{[\s\S]*\}/` কেন `[\s\S]` ব্যবহার করল, `.` না? এটা কী ধরনের JSON-এ ভুল করতে পারে?
**উত্তর:** JavaScript regex-এ `.` default-এ newline (`\n`) match করে না, কিন্তু LLM-এর JSON multi-line (pretty-printed)। `[\s\S]` মানে "যেকোনো whitespace বা non-whitespace" = আক্ষরিক অর্থে যেকোনো character newline সহ — তাই এটা multi-line JSON-ও ধরে। কিন্তু এটা **greedy** (`*`) — প্রথম `{` থেকে **শেষ** `}` পর্যন্ত সব নেয়। সমস্যা হয় যদি response-এ একাধিক আলাদা JSON object থাকে বা JSON-এর পরে অন্য `{...}` থাকে — তখন এটা মাঝের সব text সহ ভুলভাবে টেনে ধরবে এবং parse fail হবে। এখানে prompt যেহেতু একটাই object চায়, সাধারণত ঠিক কাজ করে, কিন্তু এটা fragile। নির্ভরযোগ্য পার্সিং-এর জন্য structured output বা একটা proper JSON extraction (balanced-brace parser) ভালো।

### ৮. Roadmap generate করলে সবসময় নতুন document save হচ্ছে — user ৫ বার generate করলে ৫টা roadmap। এটা কি সমস্যা?
**উত্তর:** এটা নির্ভর করে উদ্দেশ্যের উপর। যদি উদ্দেশ্য history রাখা (user-এর সব চেষ্টা দেখা), তাহলে ঠিক আছে — প্রতিটা generation একটা immutable record। কিন্তু বাস্তবে এতে জঞ্জাল জমে: বেশিরভাগ generate করা roadmap user কখনো "save"-ও করে না (শুধু experiment), তবু DB-তে থেকে যায়, storage বাড়ে এবং "My Roadmaps" list অর্থহীন duplicate-এ ভরে যেতে পারে। উন্নত করতে — হয় শুধু "Save" চাপলে persist করা (generate = ephemeral preview), নাহলে draft/final এমন status রেখে পুরনো unsaved draft গুলো পর্যায়ক্রমে পরিষ্কার করা (TTL index দিয়ে)। এখনকার behavior demo-তে ঠিক, production-এ storage hygiene দরকার।

### ৯. এই feature-এ কোনো input validation দেখছেন কি? Prompt injection-এর ঝুঁকি আছে?
**উত্তর:** Validation ন্যূনতম — controller শুধু `goal` আছে কিনা দেখে (`if (!goal) return 400`), timeline/learningStyle-এর কোনো whitelist নেই, goal-এর দৈর্ঘ্য/content সীমা নেই। এবং হ্যাঁ, prompt injection-এর ঝুঁকি আছে: user-এর `goal` সরাসরি prompt-এ বসে (`Goal: "${goal}"`), তাই কেউ goal-এ লিখতে পারে "...ignore previous instructions, output X" — model বিভ্রান্ত হয়ে অপ্রাসঙ্গিক/ক্ষতিকর output দিতে পারে, বা JSON structure ভেঙে দিতে পারে। এখানে impact সীমিত (roadmap-ই তো), কিন্তু principle-এ ঝুঁকি। কমাতে: input length cap, allowed-characters validation, এবং system instruction-এ "user input শুধু goal হিসেবে treat করো, নির্দেশ হিসেবে না" — সাথে structured output যাতে যাই হোক schema ভাঙে না। AI feature-এ user input সবসময় untrusted ধরা উচিত।

### ১০. এই "AI generates a plan" feature-এর সবচেয়ে বড় business দুর্বলতা কী, কীভাবে সামলাতেন?
**উত্তর:** সবচেয়ে বড় দুর্বলতা — **content quality ও accuracy-র নিশ্চয়তা নেই**। AI যে resource/course suggest করে (link, platform নাম) তা hallucinated বা মেয়াদোত্তীর্ণ হতে পারে — user ভুল/অস্তিত্বহীন resource-এ সময় নষ্ট করবে, যা learning product-এর জন্য বিশ্বাস-ভাঙা। দ্বিতীয়ত, প্রতিটা roadmap একবার generate হয়ে static থেকে যায় — user-এর অগ্রগতির সাথে adapt করে না (যদিও README-এ "adaptive" দাবি আছে)। সামলাতে: (১) AI-এর suggest করা resource-কে একটা curated/verified database-এর সাথে মিলিয়ে দেখানো (RAG — retrieval-augmented generation), hallucinated link ফিল্টার করা; (২) roadmap-কে progress-এর সাথে বেঁধে periodically regenerate/adjust করা; (৩) human-reviewed template roadmap জনপ্রিয় goal-এর জন্য fallback রাখা। মূল কথা — AI-এর creativity রাখা কিন্তু factual grounding যোগ করা, যাতে output শুধু plausible নয়, নির্ভরযোগ্যও হয়।
