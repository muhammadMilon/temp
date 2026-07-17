# 09 — AI Quiz: Generation & Secure Server-Side Evaluation

## এই ফিচারটা আসলে কী?

এটা SkillVoyager-এর সবচেয়ে জটিল ও শিক্ষণীয় feature — এটা AI, security, আর gamification একসাথে মেশায়। User একটা **topic** (যেমন "React Hooks"), difficulty আর প্রশ্ন সংখ্যা (৫ বা ১০) দেয়; Gemini একটা quiz বানায় — প্রতিটা প্রশ্নে ৪টা option, সঠিক উত্তর, আর ব্যাখ্যা।

সবচেয়ে গুরুত্বপূর্ণ design সিদ্ধান্ত — **সঠিক উত্তর frontend-এ পাঠানো হয় না**। Quiz বানানোর সময় correct answer DB-তে save হয়, কিন্তু user-এর কাছে শুধু প্রশ্ন ও option যায়। User answer submit করলে **server-side** এ মিলিয়ে score দেওয়া হয়। এটাই "secure evaluation" — user browser-এ উত্তর দেখে cheat করতে পারে না।

Score-এর সাথে আছে reward loop: প্রতি সঠিক উত্তরে **+১০ points**, আর quiz-এর topic অনুযায়ী **skill radar-এ boost** ([05 দেখুন](05-progress-dashboard-and-radar.md))। মানে quiz দেওয়া সরাসরি user-এর progress ও leaderboard-এ প্রতিফলিত হয়।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| AI Service | `backend/services/quiz.service.js` | Gemini দিয়ে strict-JSON quiz বানায় |
| Controller | `backend/controllers/quiz.controller.js` | generate (answer strip) + evaluate (score) |
| Routes | `backend/routes/quiz.routes.js` | `/generate`, `/submit` |
| DB Model | `backend/models/Quiz.js`, `backend/models/QuizResult.js` | quiz ও result সংরক্ষণ |
| UI | `frontend/src/pages/Quiz/QuizSession.jsx` | প্রশ্ন দেখায়, answer জমা করে submit করে |
| UI | `.../Quiz/QuizGenerator.jsx`, `QuizResult.jsx` | quiz শুরু ও ফলাফল দেখায় |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`quiz.service.js`** — একটা কড়া `systemInstruction` দিয়ে Gemini-কে exact JSON schema-তে quiz বানাতে বলে, markdown fence পরিষ্কার করে, `JSON.parse` করে, parse-fail-কেও retry করে।
- **`quiz.controller.js`** — feature-এর brain: `generateQuiz` (save করে কিন্তু correct answer বাদ দিয়ে পাঠায়), `evaluateQuiz` (server-side score, points, skill boost)।
- **`Quiz.js` / `QuizResult.js`** — quiz (correct_answer সহ) আর প্রতিটা attempt-এর result schema।
- **`QuizSession.jsx`** — user প্রশ্নের উত্তর দেয়, সব উত্তর না দিলে submit আটকায়, তারপর `/submit`-এ পাঠায়।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — AI-কে strict JSON schema-তে quiz বানানো

`quiz.service.js`-এর `systemInstruction` AI-কে exact structure বলে দেয় — এবং জোর দেয় "correct answer অবশ্যই দাও, কিন্তু system পরে evaluate করবে"।

```js
// backend/services/quiz.service.js  (systemInstruction-এর অংশ)
// 4. The correct answer MUST be included in the JSON but clearly marked ...
// 7. The response MUST be valid JSON only. Do not add extra text, no markdown block syntax.
// JSON structure: { quiz_title, total_questions, questions: [ { id, question,
//   options: {A,B,C,D}, correct_answer: "A/B/C/D", explanation } ] }

const result = await model.generateContent(prompt);
let text = result.response.text();
text = text.replace(/^```json\s*/, '').replace(/^```\s*/, '').replace(/```\s*$/, '').trim();  // fence পরিষ্কার
return JSON.parse(text);
```

### ধাপ ২ — Generate: save করো, কিন্তু answer বাদ দিয়ে পাঠাও (নিরাপত্তার হৃদয়)

Quiz DB-তে সম্পূর্ণ (correct answer সহ) save হয়, কিন্তু frontend-এ শুধু প্রশ্ন ও option যায় — correct answer **ইচ্ছাকৃতভাবে বাদ**।

```js
// backend/controllers/quiz.controller.js
const newQuiz = new Quiz({ quiz_title: aiQuiz.quiz_title, topic, skillLevel,
    total_questions: aiQuiz.total_questions, questions: aiQuiz.questions });   // full, answer সহ
await newQuiz.save();

const transformedQuestions = newQuiz.questions.map(q => ({
    question: q.question,
    options: [q.options.A, q.options.B, q.options.C, q.options.D],
    // ⬅️ correct_answer এখানে নেই — নিরাপত্তার জন্য পাঠানো হচ্ছে না
}));
res.status(200).json({ quizId: newQuiz._id, topic, skillLevel, questions: transformedQuestions, /* ... */ });
```

গুরুত্বপূর্ণ comment কোডেই আছে: *"We don't send the correct_answer to the frontend here for security"*।

### ধাপ ৩ — User উত্তর দেয় (frontend)

`QuizSession` প্রতিটা প্রশ্নের নির্বাচিত option একটা array-তে রাখে, সব উত্তর না দিলে submit block করে।

```jsx
// frontend/src/pages/Quiz/QuizSession.jsx
const handleSubmit = async () => {
    if (answers.includes('')) { toast.warning('Please answer all questions...'); return; }
    const response = await fetch(`${API_BASE}/api/quiz/submit`, {
        method: 'POST',
        body: JSON.stringify({ userEmail: user?.email || 'anonymous', uid: user?.uid, quizId, answers })
    });
    // ... result পেয়ে /quiz/result-এ navigate
};
```

### ধাপ ৪ — Evaluate: server-side scoring

Submit এলে server DB থেকে আসল quiz (answer সহ) এনে user-এর answer মিলিয়ে দেখে। User string পাঠায় (option-এর text), server সেটাকে letter-এ map করে correct_answer-এর সাথে মেলায়।

```js
// backend/controllers/quiz.controller.js
const quiz = await Quiz.findById(quizId);        // ⬅️ আসল answer DB থেকে, client থেকে না
let score = 0;
quiz.questions.forEach((q, index) => {
    const userAnswerString = answers[index];
    let userAnswerLetter = null;
    if (userAnswerString === q.options.A) userAnswerLetter = 'A';
    else if (userAnswerString === q.options.B) userAnswerLetter = 'B';
    // ... C, D
    const isCorrect = userAnswerLetter === q.correct_answer;   // ⬅️ server-side মিলানো
    if (isCorrect) score += 1;
    evaluation.push({ question: q.question, userAnswer: userAnswerString,
        correctAnswer: q.options[q.correct_answer], isCorrect, explanation: q.explanation });
});
const percentage = Math.round((score / totalQuestions) * 100);
const pointsEarned = score * 10;
```

### ধাপ ৫ — Reward: skill boost + XP

Result save করে, skill radar-এ boost দেয় (topic → skill key), আর user-এর points বাড়ায়।

```js
// backend/controllers/quiz.controller.js
if (uid) {
    const userProgress = await Progress.findOne({ uid });
    if (userProgress) {
        const skillKey = quiz.topic.toLowerCase().split(' ')[0].replace(/[^a-z]/g, '');  // "React Hooks" → "react"
        const boost = Math.round((score / totalQuestions) * 15);   // পারফরম্যান্স অনুযায়ী boost
        const currentVal = userProgress.skillStrength.get(skillKey) || 0;
        userProgress.skillStrength.set(skillKey, Math.min(100, currentVal + boost));
        await userProgress.save();
    }
}
if (userEmail && userEmail !== 'anonymous') {
    await User.findOneAndUpdate({ email: userEmail }, { $inc: { points: pointsEarned } }, { new: true });
}
res.status(200).json({ score, totalQuestions, percentage, pointsEarned, evaluation });
```

লক্ষ্য করুন — `evaluation`-এ এখন correct answer + explanation ফেরত যায়, কারণ user submit করার **পরে** সেটা দেখানো নিরাপদ ও শিক্ষণীয়।

## মূল ধারণা (Key Concepts)

- **Server-side authority** — scoring/সত্য সবসময় server-এ; client-কে বিশ্বাস করা যায় না।
- **Sensitive data withholding** — correct answer submit-এর আগ পর্যন্ত client-এ যায় না।
- **Two-phase interaction** — generate (প্রশ্ন) ও submit (মূল্যায়ন) আলাদা, state DB-তে (`quizId`)।
- **String→letter mapping** — user-এর option text-কে canonical answer key-তে রূপান্তর।
- **Performance-based reward** — score অনুযায়ী skill boost/points, flat না।
- **Strict-JSON prompting + cleanup** — AI output-কে parse-যোগ্য বানানো।
- **Post-submit reveal** — মূল্যায়নের পর correct answer + explanation দেখিয়ে শেখানো।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. Correct answer frontend-এ না পাঠিয়ে server-এ evaluate করলেন কেন? না করলে ঠিক কী ভাঙত?
**উত্তর:** কারণ client কখনো trusted না — browser-এ যা যায় তা user (বা যেকোনো script) দেখতে/বদলাতে পারে। যদি correct answer quiz-এর সাথে frontend-এ পাঠাতাম, তাহলে যে কেউ Network tab বা React DevTools খুলে প্রতিটা প্রশ্নের সঠিক উত্তর দেখে ১০০% স্কোর করত — quiz-এর পুরো উদ্দেশ্য (শেখা যাচাই) নষ্ট, আর points/leaderboard অর্থহীন হয়ে যেত। এমনকি frontend-এ evaluate করলে user সরাসরি `/quiz/submit`-এ মনগড়া score পাঠাতে পারত। Server-side evaluation নিশ্চিত করে — সত্যের একমাত্র উৎস server, যেখানে user-এর হাত নেই। এটা security-র মৌলিক নীতি: **never trust the client**।

### ২. এখনো কি এই quiz সম্পূর্ণ cheat-proof? কোন ফাঁক আছে?
**উত্তর:** না, একটা বড় ফাঁক আছে — `evaluateQuiz` endpoint-এ **কোনো authentication/verification নেই**। User `uid`, `userEmail`, `quizId`, `answers` সব body-তে পাঠায়, server verify করে না যে এই request সত্যিই সেই user-এর কিনা। তাই কেউ চাইলে: (ক) একই quiz বারবার submit করে সবচেয়ে ভালো score না পাওয়া পর্যন্ত চেষ্টা করতে পারে (কোনো attempt limit নেই), (খ) অন্যের `userEmail`/`uid` পাঠিয়ে তার points বাড়াতে/কমাতে পারে। এছাড়া correct answer লুকানো থাকলেও একই quiz একাধিকবার generate/submit করে answer বের করা যায়। শক্ত করতে: ID token verify করে uid নেওয়া, প্রতি quiz-এ attempt limit/one-time-submit, আর একই quizId-এ duplicate submit reject করা দরকার।

### ৩. User option-এর **text** পাঠায় (letter না) — server সেটাকে letter-এ map করে। এই design-এর দুর্বলতা কী?
**উত্তর:** এটা কাজ করে, কিন্তু fragile। Server user-এর পাঠানো string-কে `q.options.A/B/C/D`-এর সাথে **exact string match** করে letter বের করে। সমস্যা: যদি option text-এ কোনো unicode/whitespace/encoding পার্থক্য থাকে (frontend trim করল, বা special character ভিন্নভাবে render হলো), তাহলে match fail করে `userAnswerLetter` null থেকে যাবে — সঠিক উত্তরও ভুল গণ্য হবে। আরও, দুটো option-এর text হুবহু এক হলে (AI মাঝে মাঝে duplicate দেয়) mapping অস্পষ্ট হয়। অনেক পরিষ্কার হতো — frontend সরাসরি letter/index পাঠাত ('A' বা 0), text না; তখন কোনো string-matching লাগত না, শুধু `answers[index] === q.correct_answer`। Text পাঠানোর একমাত্র সুবিধা frontend-কে option order জানতে হয় না, কিন্তু সেটা index দিয়েও সমাধান হতো।

### ৪. Quiz generate ও submit — দুটো আলাদা request; মাঝের state কোথায় থাকে? এর সুবিধা-অসুবিধা?
**উত্তর:** State থাকে **DB-তে**, `quizId` দিয়ে। Generate করলে quiz (answer সহ) save হয়ে `quizId` ফেরত আসে; submit করলে সেই `quizId` দিয়ে আসল quiz আবার লোড করে evaluate হয়। সুবিধা — server stateless থাকে (কোনো session memory লাগে না), যা serverless (Vercel)-এ জরুরি; আর evaluation-এর সময় authoritative answer DB-তেই আছে, client-কে বিশ্বাস করতে হয় না। অসুবিধা — প্রতিটা generate একটা DB write, আর evaluate আরেকটা read; আর অসম্পূর্ণ/পরিত্যক্ত quiz (generate করে submit না করা) DB-তে জমে। এছাড়া `quizId` client-এর হাতে, তাই কেউ একটা পুরনো quizId নিয়ে বারবার খেলতে পারে (attempt limit না থাকায়)। State-কে DB-তে রাখা এই architecture-এ সঠিক পছন্দ, তবে cleanup ও attempt-control দরকার।

### ৫. Skill boost-এ `quiz.topic.toLowerCase().split(' ')[0]` দিয়ে key বানানো হচ্ছে — এতে কী সমস্যা?
**উত্তর:** এটা topic-এর শুধু **প্রথম শব্দ** নেয় — "React Hooks" → `react`, "Machine Learning" → `machine`, "Data Structures" → `data`। সমস্যা: (ক) দুই-শব্দের topic-এর দ্বিতীয় অংশ হারায় — "Machine Learning" আর "Machine Vision" দুটোই `machine` key-তে boost দেবে, ভুলভাবে মিশে যায়; (খ) এই key onboarding/progress-এর skill key-এর সাথে নাও মিলতে পারে — onboarding "Node.js" কে `nodejs` করে, কিন্তু quiz topic "Node Basics" কে `node` করবে, ফলে boost ভিন্ন key-তে গিয়ে radar-এ প্রতিফলিত না-ও হতে পারে (silent mismatch)। এটা [05](05-progress-dashboard-and-radar.md)-এ আলোচিত normalization inconsistency-র একটা বাস্তব উদাহরণ। সমাধান — সব জায়গায় একটা shared `normalizeSkillKey()` ব্যবহার করা এবং topic থেকে সরাসরি একটা canonical skill map করা, প্রথম-শব্দ heuristic না।

### ৬. Boost `Math.round((score/total) * 15)` — flat না দিয়ে performance-based করলেন কেন?
**উত্তর:** কারণ শেখার প্রমাণ score-এ। যদি quiz সম্পন্ন করলেই flat boost দিতাম, তাহলে সব উত্তর ভুল দিয়েও কেউ skill "বেড়েছে" দেখত — যা মিথ্যা ও gaming-প্রবণ। `score/total` অনুপাত দিয়ে গুণ করায় — ১০০% পেলে পূর্ণ boost (~১৫), ৫০% পেলে অর্ধেক (~৭.৫→৮), ০% পেলে boost নেই — অর্থাৎ skill radar আসলেই দক্ষতার প্রতিফলন করে, শুধু অংশগ্রহণের না। এটা reward-কে outcome-এর সাথে বাঁধে, যা সৎ gamification-এর মূল নীতি। (তুলনায় points `score * 10`-ও performance-based, তাই দুটোই consistent।) দুর্বলতা — একই topic বারবার quiz দিয়ে ধীরে ধীরে ১০০ পর্যন্ত farm করা যায়, তাই attempt/cooldown control এখানেও দরকার।

### ৭. AI যদি invalid JSON বা markdown-mোড়ানো quiz দেয়, `quiz.service.js` কীভাবে সামলায়?
**উত্তর:** তিন স্তরে: (১) `systemInstruction`-এ স্পষ্টভাবে বলা "valid JSON only, no markdown block syntax" — model-কে আগেই সঠিক format-এ ঠেলা; (২) তবু model যদি ` ```json ` fence দেয়, `text.replace(...)` দিয়ে সেই fence গুলো ছেঁটে ফেলা হয় parse-এর আগে; (৩) `JSON.parse` তবু fail করলে (`SyntaxError`), সেটাকে **retryable** ধরা হয়েছে — `isRetryable`-এ `error.name === "SyntaxError"` আছে — তাই আবার Gemini কল করে নতুন উত্তর নেওয়া হয় (৩ বার পর্যন্ত)। এই "parse fail = retry" বুদ্ধিদীপ্ত, কারণ LLM non-deterministic — এক চেষ্টায় নোংরা দিলে পরের চেষ্টায় পরিষ্কার দিতে পারে। সবচেয়ে robust হতো Gemini-এর structured-output/responseSchema, কিন্তু এই cleanup+retry combo বাস্তবে ভালো কাজ করে।

### ৮. `evaluateQuiz`-এ correct answer ও explanation response-এ ফেরত পাঠানো হচ্ছে — এটা কি ধাপ ২-এর security নীতির বিরোধী?
**উত্তর:** না, বরং এটা সঠিক timing-এর উদাহরণ। নীতিটা ছিল — user উত্তর **দেওয়ার আগে** correct answer লুকানো, যাতে cheat না করতে পারে। কিন্তু submit করার **পরে** correct answer ও explanation দেখানো শিক্ষার জন্য অপরিহার্য — user জানতে চায় কোথায় ভুল করল ও কেন। এই "post-submit reveal" quiz-এর learning value-র মূল অংশ। মূল বিষয় হলো *কখন* data প্রকাশ করা হচ্ছে: মূল্যায়নের আগে নয়, পরে। তাই এটা নীতির বিরোধী নয় — বরং নীতিটার সঠিক প্রয়োগ: sensitive data শুধু তখনই দাও যখন সেটা প্রকাশ করা নিরাপদ (এখানে, উত্তর জমা দেওয়ার পর)।

### ৯. একই generate-throw-away pattern — প্রতিটা quiz DB-তে save হয়। Scale-এ এটা কী সমস্যা করবে?
**উত্তর:** প্রতিটা "generate" একটা নতুন `Quiz` document তৈরি করে, submit হোক বা না হোক। Scale-এ — হাজার user দিনে একাধিক quiz generate করলে `Quiz` collection দ্রুত ফুলে যাবে, যার বেশিরভাগ একবার ব্যবহার হয়ে পরিত্যক্ত। এতে storage খরচ বাড়ে ও কোনো cleanup নেই। এছাড়া `QuizResult`-ও প্রতি submit-এ জমে। সমাধান: (ক) পরিত্যক্ত/পুরনো quiz-এ **TTL index** দিয়ে অটো-মেয়াদ (যেমন ৭ দিন পর মুছে যাবে যদি কোনো result না থাকে), (খ) জনপ্রিয় topic-এর quiz cache/reuse করা যাতে প্রতিবার Gemini + save না লাগে, (গ) analytics-এর জন্য শুধু QuizResult রেখে ব্যবহৃত-হয়ে-যাওয়া প্রশ্ন archive করা। এখন demo scale-এ ঠিক, কিন্তু growth-এ retention policy দরকার।

### ১০. এই feature-টা AI + security + gamification মেশায় — এখানে সবচেয়ে বড় স্থাপত্যগত (architectural) ঝুঁকি কী?
**উত্তর:** সবচেয়ে বড় ঝুঁকি — **একটা endpoint অনেক দায়িত্ব নিচ্ছে এবং সেগুলোর মধ্যে atomicity/verification নেই**। `evaluateQuiz` একসাথে: quiz লোড করে, score করে, QuizResult save করে, Progress skillStrength বদলে save করে, আর User points `$inc` করে — চারটা আলাদা write, কোনো transaction ছাড়া। মাঝপথে একটা fail করলে (যেমন points বাড়ল কিন্তু result save হলো না, বা উল্টো) inconsistent state হয়। তার উপর কোনো auth/attempt-limit না থাকায় এই reward path abuse-প্রবণ (একই quiz বারবার submit করে points/skill farm)। ঝুঁকি কমাতে: (১) responsibility ভাগ করা — evaluate pure রাখা, reward একটা আলাদা verified step-এ; (২) auth + idempotency (একই quizId-user একবারই score পাবে); (৩) সমালোচনামূলক write গুলো transaction-এ মোড়া বা eventual consistency মেনে best-effort করা; (৪) rate limit। মূলত — যেখানে AI (unpredictable), reward (abusable), আর multi-write (inconsistency-prone) একত্র হয়, সেখানে verification ও idempotency সবচেয়ে জরুরি।
