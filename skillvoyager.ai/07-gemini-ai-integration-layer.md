# 07 — Gemini AI Integration Layer (Retry + Key Rotation)

## এই ফিচারটা আসলে কী?

SkillVoyager-এর "AI" অংশ — skill gap analysis, roadmap generation, quiz, mentor chat — সবই Google-এর **Gemini** model কল করে চলে। এখন প্রশ্ন হলো, প্রতিটা feature যদি আলাদা করে Gemini SDK setup করত, তাহলে model নাম, API key, retry logic, error handling সব জায়গায় duplicate হতো এবং একটা বদলাতে গেলে সব ঘাঁটতে হতো।

তাই এখানে একটা **centralized integration layer** বানানো হয়েছে — `backend/services/gemini.service.js` — যেটা একটা মাত্র function `callGemini(prompt, systemInstruction, history)` export করে। সব AI feature এই এক দরজা দিয়ে Gemini-তে যায়।

এই layer-এর দুটো বুদ্ধিমান দিক আছে: (১) **Retry with backoff** — Gemini overloaded (503) বা rate-limited (429) হলে সাথে সাথে fail না করে বেড়ে-চলা delay দিয়ে কয়েকবার চেষ্টা করে; (২) **API key rotation/fallback** — একাধিক env key-এর মধ্যে যেটা আগে পাওয়া যায় সেটা ব্যবহার করে, quota ভাগ করা বা fallback রাখার জন্য।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| Core Service | `backend/services/gemini.service.js` | `callGemini()` — retry + key + chat/one-shot |
| Consumer | `backend/controllers/analysis.controller.js` | skill gap analysis-এ `callGemini` কল করে |
| Consumer | `backend/controllers/roadmap.controller.js` | roadmap-এ `callGemini` কল করে |
| Consumer | `backend/controllers/mentor.controller.js` | mentor chat-এ history সহ `callGemini` কল করে |
| Sibling | `backend/services/quiz.service.js` | quiz-এর জন্য আলাদা but একই প্যাটার্নের Gemini call |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`gemini.service.js`** — একমাত্র জায়গা যেখানে model নাম, key selection ও retry policy থাকে; সব AI feature এর উপর নির্ভর করে (single point of change)।
- **`analysis/roadmap/mentor.controller.js`** — এরা "consumer": শুধু একটা prompt (আর দরকারে systemInstruction/history) বানিয়ে `callGemini`-কে দেয়, নিজেরা SDK জানে না।
- **`quiz.service.js`** — quiz-এর জন্য প্রায় একই retry প্যাটার্ন কিন্তু আলাদা key ও strict-JSON systemInstruction (কেন আলাদা, দেখুন প্রশ্ন ৯)।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — API key rotation/fallback

Service load হওয়ার সময় একাধিক সম্ভাব্য env key-এর মধ্যে প্রথম যেটা পাওয়া যায় সেটা নেয়। এতে ভিন্ন feature ভিন্ন key দিয়ে চলতে পারে, বা একটা key শেষ হলে অন্যটা fallback থাকে।

```js
// backend/services/gemini.service.js
const { GoogleGenerativeAI } = require("@google/generative-ai");

const apiKey = process.env.GEMINI_API_KEY_roadmap
            || process.env.GEMINI_API_KEY_milon
            || process.env.GEMINI_API_KEY;   // ⬅️ প্রথম উপলব্ধ key
const genAI = new GoogleGenerativeAI(apiKey);
```

### ধাপ ২ — Model + optional system instruction

`callGemini` model instance বানায়, আর দরকার হলে একটা `systemInstruction` (AI-এর persona/নিয়ম) সেট করে।

```js
// backend/services/gemini.service.js
const callGemini = async (prompt, systemInstruction = "", history = []) => {
  const modelName = "gemini-2.5-flash-lite";       // দ্রুত ও সস্তা variant
  const model = genAI.getGenerativeModel({
    model: modelName,
    systemInstruction: systemInstruction || undefined,   // খালি হলে দেওয়াই হয় না
  });
  // ...
};
```

### ধাপ ৩ — One-shot বনাম chat (history থাকলে)

একই function দুই mode সামলায় — `history` থাকলে multi-turn chat (mentor), না থাকলে single generate (skill gap/roadmap)।

```js
// backend/services/gemini.service.js
if (history.length > 0) {
    const chat = model.startChat({ history });         // context-সহ কথোপকথন
    const result = await chat.sendMessage(prompt);
    return result.response.text();
} else {
    const result = await model.generateContent(prompt);  // এককালীন প্রশ্ন
    return result.response.text();
}
```

### ধাপ ৪ — Retry with exponential-ish backoff

সবচেয়ে গুরুত্বপূর্ণ অংশ। Gemini মাঝে মাঝে 503 (overloaded) বা 429 (rate limit) দেয় — এগুলো transient, একটু পরে চেষ্টা করলে সফল হয়। তাই retryable error হলে বেড়ে-চলা delay দিয়ে সর্বোচ্চ ৩ বার চেষ্টা।

```js
// backend/services/gemini.service.js
let lastError;
const maxRetries = 3;
for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
        // ... (ধাপ ৩ এর call, সফল হলে return)
    } catch (error) {
        lastError = error;
        const isRetryable = error.message.includes("503")
                         || error.message.includes("high demand")
                         || error.message.includes("429");
        if (isRetryable && attempt < maxRetries) {
            const delay = attempt * 2000;   // ⬅️ 2s, 4s, 6s — বেড়ে চলা wait
            console.warn(`Gemini (Attempt ${attempt}) failed: ${error.message}. Retrying in ${delay}ms...`);
            await new Promise(resolve => setTimeout(resolve, delay));
            continue;
        }
        break;   // non-retryable বা শেষ চেষ্টা → বেরিয়ে যাও
    }
}
throw lastError;   // সব চেষ্টা ব্যর্থ হলে caller-কে জানাও
```

গুরুত্বপূর্ণ: শুধু transient error-এ retry হয়; permanent error (যেমন invalid prompt, auth fail) হলে সাথে সাথে থামে — অকারণে wait করে না।

### ধাপ ৫ — Consumer কীভাবে ব্যবহার করে

Consumer শুধু prompt বানায়, তারপর `callGemini`-কে দেয় — retry/key/SDK-এর কিছুই জানতে হয় না।

```js
// backend/controllers/analysis.controller.js
const responseText = await callGemini(promptTemplate);      // one-shot
// backend/controllers/mentor.controller.js
const aiText = await callGemini(message, systemPrompt, history);   // chat, context সহ
```

## মূল ধারণা (Key Concepts)

- **Facade / single integration point** — সব AI call এক function দিয়ে; SDK-এর জটিলতা লুকানো।
- **Retry with backoff** — transient failure-এ বেড়ে-চলা delay দিয়ে পুনঃচেষ্টা, resilience বাড়ায়।
- **Retryable vs. permanent error** — শুধু 503/429-এ retry, বাকিতে দ্রুত fail।
- **API key fallback/rotation** — একাধিক key-এর মধ্যে প্রথম উপলব্ধটা, quota resilience।
- **Overloaded polymorphic function** — একই `callGemini` chat ও one-shot দুটোই সামলায়।
- **System instruction** — model-এর persona/নিয়ম প্রতি call-এ inject করা।
- **Separation of concerns** — controller "কী জিজ্ঞেস করব" জানে, service "কীভাবে জিজ্ঞেস করব"।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. AI call-কে একটা centralized service-এ রাখলেন কেন, প্রতিটা controller-এ সরাসরি SDK কল না করে?
**উত্তর:** DRY এবং single point of change-এর জন্য। skill gap, roadmap, mentor — সবাই Gemini কল করে; প্রতিটা যদি নিজে SDK setup, key selection, retry, error handling করত, তাহলে (ক) কোড চারগুণ duplicate হতো, (খ) model নাম বা retry policy বদলাতে চারটা ফাইল ঘাঁটতে হতো, (গ) একটায় bug fix অন্যটায় থেকে যেত। `callGemini` একটা facade — সব controller শুধু prompt দেয়, "কীভাবে Gemini-তে যাব" পুরোটা এক জায়গায়। ফলে model upgrade (যেমন নতুন Gemini version), retry tune, বা key যোগ করা — সব এক ফাইলে হয়। এটা maintainability-র জন্য textbook decision।

### ২. Retry-তে exponential backoff (বা এখানে linear `attempt * 2000`) কেন, সাথে সাথে বারবার retry না করে?
**উত্তর:** কারণ 503/429 মানে সার্ভার এখন overloaded বা আপনি rate limit ছুঁয়েছেন — সাথে সাথে আবার আঘাত করলে সমস্যা আরও বাড়ে (thundering herd), সফল হওয়ার সম্ভাবনা কম। একটু অপেক্ষা করলে load কমে বা rate window খুলে যায়, তখন retry সফল হয়। এখানে delay `attempt * 2000` (2s, 4s, 6s) — প্রতি চেষ্টায় বেশি অপেক্ষা, যাতে সার্ভারকে recover করার সময় দেওয়া যায়। বিশুদ্ধ exponential হতো `2^attempt`, আর আদর্শে random "jitter" যোগ করা হয় (যাতে সব client একসাথে retry না করে) — এখানে linear ও jitter-বিহীন, যা simple কিন্তু jitter থাকলে আরও ভালো হতো।

### ৩. সব error-এ retry না করে শুধু 503/429/"high demand"-এ retry — এই পার্থক্যটা গুরুত্বপূর্ণ কেন?
**উত্তর:** কারণ সব error retry-যোগ্য না। 503 (overloaded) ও 429 (rate limit) হলো **transient** — পরিস্থিতি বদলালে ঠিক হয়ে যায়, তাই retry অর্থবহ। কিন্তু 400 (invalid prompt), 401/403 (ভুল/মেয়াদোত্তীর্ণ API key), বা safety-block — এগুলো **permanent**; একই request আবার পাঠালে একই ভুল হবে, শুধু user-কে অপেক্ষা করানো ও quota নষ্ট করা ছাড়া কিছু হবে না। তাই কোড error message-এ 503/429 আছে কিনা দেখে সিদ্ধান্ত নেয় — transient হলে retry, নাহলে সাথে সাথে `throw`। এটা resilience আর দ্রুত-fail-এর মধ্যে সঠিক ভারসাম্য।

### ৪. Error detection হচ্ছে `error.message.includes("503")` দিয়ে — এই string-matching approach-এর দুর্বলতা কী?
**উত্তর:** এটা fragile। SDK বা API যদি error message-এর wording বদলায় (যেমন "503" না লিখে অন্যভাবে জানায়), তাহলে retryable error আর ধরা পড়বে না — retry logic নীরবে ভেঙে যাবে। String-matching semantic না, শুধু আশা করা হচ্ছে বিশেষ substring থাকবে। ভালো approach হতো error-এর structured field ব্যবহার — যেমন `error.status` বা `error.code` (HTTP status code) দেখা, message parse না করে। অনেক SDK error object-এ status attach করে; সেটা থাকলে `[429, 503].includes(error.status)` অনেক নির্ভরযোগ্য হতো। এটা একটা সচেতন dev-এর ধরার মতো সূক্ষ্মতা।

### ৫. একাধিক API key-এর fallback (`_roadmap || _milon || GEMINI_API_KEY`) — এর উদ্দেশ্য কী, আর এটা কি সত্যিকারের "rotation"?
**উত্তর:** উদ্দেশ্য দুটো — (ক) quota resilience: Gemini free tier-এ প্রতি key-এ rate/quota সীমা থাকে; একাধিক key থাকলে ভিন্ন feature ভিন্ন key ব্যবহার করে load ভাগ করতে পারে, (খ) fallback: primary key না থাকলে (env-এ set না থাকলে) পরেরটা ব্যবহার হয়। তবে সততার সাথে — এটা সত্যিকারের "rotation" না। Rotation মানে runtime-এ key-এর মধ্যে ঘোরাফেরা বা একটা 429 দিলে পরেরটায় switch করা। এখানে load-time-এ **একবার** একটা key বেছে নেওয়া হয়, তারপর সেটাই সব request-এ ব্যবহৃত হয় — একটা key rate-limit হলে অন্যটায় স্বয়ংক্রিয়ভাবে যায় না। সত্যিকারের rotation করতে হলে key-এর array রেখে round-robin বা 429-এ next-key retry করতে হতো।

### ৬. `history` থাকা/না-থাকার উপর ভিত্তি করে chat বনাম one-shot — এই দুই mode এক function-এ রাখা কি ভালো design?
**উত্তর:** এখানে যুক্তিসঙ্গত, কারণ দুই mode-এর ৯০% (key, model, retry, error handling) একই — শুধু শেষ call-টা ভিন্ন (`startChat().sendMessage` বনাম `generateContent`)। এক function-এ রাখায় retry/key logic duplicate হয় না, caller শুধু history দিলে multi-turn পায়। এটা pragmatic। দুর্বলতা — function-এর behavior parameter-এর উপর নির্ভর করে (mildly "control coupling"), এবং history mode-এ system instruction/streaming নিয়ে ভবিষ্যতে জটিলতা বাড়লে দুটো আলাদা function (`generate` ও `chat`) পরিষ্কার হতো। এখনকার scale-এ এক function ঠিক আছে; branching বাড়লে ভাগ করা উচিত।

### ৭. যদি Gemini পুরোপুরি down থাকে (তিন retry-ই fail), user-এর কী অভিজ্ঞতা হয়? আরও ভালো করা যেত?
**উত্তর:** তিন চেষ্টার পর `throw lastError` হয়, controller সেটা `catch` করে 500 + error message দেয়, frontend toast-এ "Failed to generate..." দেখায় (এবং roadmap-এর ক্ষেত্রে frontend আরও client-side retry করে)। অর্থাৎ user একটা error দেখে, কিন্তু ~১২ সেকেন্ড (2+4+6) অপেক্ষার পর — যা দীর্ঘ। উন্নত করা যেত: (ক) একটা graceful fallback — cached/template roadmap বা "AI ব্যস্ত, একটু পরে চেষ্টা করুন" সহ estimated সময়, (খ) request-টা queue-তে রেখে পরে process করে notification পাঠানো, (গ) circuit breaker — পরপর অনেক fail হলে কিছুক্ষণ AI call বন্ধ রেখে দ্রুত fallback দেওয়া, প্রতিবার ১২s অপেক্ষা না করানো। এতে outage-এও UX ভাঙে না।

### ৮. `gemini-2.5-flash-lite` model বেছে নিলেন কেন, বড়/শক্তিশালী model না?
**উত্তর:** এটা cost, speed ও use-case-এর ভারসাম্য। "flash-lite" variant দ্রুত এবং সস্তা — SkillVoyager-এর কাজগুলো (skill gap বের করা, roadmap-এর structured JSON, quiz, mentor chat) খুব গভীর reasoning চায় না, বরং দ্রুত ও consistent output চায়। User একটা roadmap-এর জন্য ৩০ সেকেন্ড অপেক্ষা করতে চায় না; lite model কম latency দেয়। বড় model বেশি সূক্ষ্ম output দিত কিন্তু বেশি খরচ ও ধীর, আর free-tier-এ quota দ্রুত শেষ হতো। যেহেতু এটা একটা learning product যেখানে অনেক user অনেক request করবে, cost-per-call কম রাখা গুরুত্বপূর্ণ — তাই lite model যুক্তিসঙ্গত। প্রয়োজনে নির্দিষ্ট feature-এ (যেমন গভীর analysis) বড় model ব্যবহার করা যেত, কিন্তু default lite।

### ৯. quiz-এর জন্য আলাদা `quiz.service.js` কেন, `gemini.service.js` reuse না করে?
**উত্তর:** কারণ quiz-এর চাহিদা আলাদা — এর জন্য একটা **strict JSON schema** এবং সেই JSON parse করা দরকার, শুধু text না। `quiz.service.js`-এ একটা বিস্তারিত `systemInstruction` আছে যা exact JSON structure নির্দিষ্ট করে, response থেকে markdown fence (```json) পরিষ্কার করে, তারপর `JSON.parse` করে — আর parse fail (SyntaxError)-কেও retryable ধরে আবার চেষ্টা করে। এটা `callGemini`-এর generic text-return interface-এর চেয়ে বেশি specialized। তবে সততার সাথে — এই আলাদা করাটা duplication তৈরি করেছে (retry loop প্রায় একই)। ভালো design হতো `callGemini`-কে একটা optional `responseSchema`/`parseJson` mode দিয়ে বাড়ানো, বা quiz service-কে callGemini-র উপর build করা — তাহলে retry logic এক জায়গায় থাকত। এখন দুই জায়গায় retry maintain করতে হয়, যা technical debt।

### ১০. এই AI layer-এ cost/abuse control নেই — production-এ কী কী যোগ করতেন?
**উত্তর:** বেশ কয়েকটা: (১) **Rate limiting per user** — একজন user মিনিটে/দিনে কতগুলো AI call করতে পারবে, নাহলে কেউ endpoint spam করে quota ও টাকা শেষ করতে পারে। (২) **Caching** — একই/অনুরূপ prompt-এর response cache করা (যেমন জনপ্রিয় roadmap goal), যাতে বারবার Gemini না ডাকতে হয়। (৩) **Input validation/size cap** — prompt-এর দৈর্ঘ্য সীমিত করা, কারণ token = খরচ; আর prompt injection ঠেকানো। (৪) **Auth on AI endpoints** — এখন endpoint গুলো verify করে না কে call করছে; login-বাধ্যতামূলক করা। (৫) **Monitoring/quota alert** — খরচ ও error rate track করা, budget ছাড়ালে alert। (৬) **Timeout** — একটা call বেশি সময় নিলে কেটে দেওয়া। এগুলো ছাড়া একটা public AI endpoint আর্থিকভাবে বিপজ্জনক — abuse হলে বড় বিল আসতে পারে।
