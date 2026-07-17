# 04 — Multi-Step Onboarding Flow

## এই ফিচারটা আসলে কী?

নতুন user প্রথমবার এলে SkillVoyager তার সম্পর্কে কিছু জানতে চায় — সে এখন কোন role-এ আছে, কী পড়ছে, কী কী skill জানে, কোন career-এ যেতে চায়, আর কত সময়ের মধ্যে। এই তথ্য একটা লম্বা form-এ একবারে চাইলে user ভয় পেয়ে ছেড়ে দেয়। তাই এটাকে ভাগ করা হয়েছে **৩টা ধাপে (multi-step wizard)** — একবারে একটা করে ছোট প্রশ্ন, উপরে progress indicator, নিচে Next/Back বাটন।

তথ্যগুলো জমা হওয়ার পর `POST /api/onboarding` এ যায়, আর backend সেটা **দুই জায়গায়** লেখে — `User.onboarding` (profile) আর `Progress` collection (learning tracking-এর শুরু)। এতে onboarding শেষ হওয়ামাত্র user-এর progress dashboard আর skill radar-এ initial data চলে আসে।

শেষ ধাপে একটা সুন্দর touch আছে — সফল হলে user তার **Roadmap PDF** আর **AI Resume PDF** সাথে সাথে download করতে পারে (`@react-pdf/renderer` দিয়ে client-side এ generate হয়)।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| UI (orchestrator) | `frontend/src/pages/Onboarding/OnboardingFlow.jsx` | পুরো wizard state, step navigation, submit |
| UI (steps) | `.../Onboarding/StepRole.jsx`, `StepSkills.jsx`, `StepCareerTimeline.jsx` | প্রতিটা ধাপের আলাদা form |
| UI (progress) | `.../Onboarding/StepIndicator.jsx` | কোন ধাপে আছে তা দেখায় |
| PDF | `.../Onboarding/RoadmapPDF.jsx`, `ResumePDF.jsx` | সফল হলে PDF generate করে |
| API | `backend/routes/onboarding.routes.js` | data নিয়ে User + Progress এ save করে |
| DB | `backend/models/User.js`, `backend/models/Progress.js` | data রাখার জায়গা |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`OnboardingFlow.jsx`** — এই ফিচারের brain: `currentStep`, `formData`, per-step validation, আর submit সব এখানে; আলাদা step component গুলোকে coordinate করে।
- **`StepRole.jsx` / `StepSkills.jsx` / `StepCareerTimeline.jsx`** — প্রতিটা "dumb" presentational component; শুধু নিজের অংশের input দেখায় আর `updateFormData` দিয়ে parent-কে জানায়।
- **`StepIndicator.jsx`** — কোন ধাপে আছি সেটা visually দেখায়, user-কে orient রাখে।
- **`onboarding.routes.js`** — skill নাম normalize করে, `User.onboarding` update করে, আর `Progress`-এ initial milestone/skillStrength বসায়।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — এক জায়গায় সব state

Parent `OnboardingFlow` একটা `formData` object-এ পাঁচটা field রাখে, আর `currentStep` দিয়ে জানে এখন কোন ধাপে আছে। এই "state উপরে, UI নিচে" প্যাটার্ন = controlled wizard।

```jsx
// frontend/src/pages/Onboarding/OnboardingFlow.jsx
const [currentStep, setCurrentStep] = useState(1);
const [formData, setFormData] = useState({ role:'', education:'', skills:[], targetCareer:'', timeline:'' });
const totalSteps = 3;

const updateFormData = (field, value) => setFormData(prev => ({ ...prev, [field]: value }));
```

### ধাপ ২ — Per-step validation (আগানোর শর্ত)

প্রতি ধাপে "Next" চাপার আগে সেই ধাপের field পূরণ হয়েছে কিনা চেক হয়। অসম্পূর্ণ হলে Next বাটন disabled থাকে।

```jsx
// frontend/src/pages/Onboarding/OnboardingFlow.jsx
const isStepValid = () => {
    switch (currentStep) {
        case 1: return formData.role && formData.education;
        case 2: return formData.skills.length >= 1;          // অন্তত একটা skill
        case 3: return formData.targetCareer && formData.timeline;
        default: return false;
    }
};
```

### ধাপ ৩ — কোন ধাপ render হবে (conditional + animation)

`currentStep`-এর মান অনুযায়ী ঠিক একটা step component দেখানো হয়, `AnimatePresence` দিয়ে ধাপ বদলের সময় slide animation।

```jsx
// frontend/src/pages/Onboarding/OnboardingFlow.jsx
<AnimatePresence mode="wait">
  <motion.div key={currentStep} initial={{opacity:0,x:20}} animate={{opacity:1,x:0}} exit={{opacity:0,x:-20}}>
    {currentStep===1 && <StepRole formData={formData} updateFormData={updateFormData}/>}
    {currentStep===2 && <StepSkills formData={formData} updateFormData={updateFormData}/>}
    {currentStep===3 && <StepCareerTimeline formData={formData} updateFormData={updateFormData}/>}
  </motion.div>
</AnimatePresence>
```

`key={currentStep}` জরুরি — এটা বদলালে React পুরনো টা unmount করে নতুন mount করে, ফলে exit/enter animation ঠিকমতো চলে।

### ধাপ ৪ — Submit → backend

শেষ ধাপে "Initialize My Voyager Profile" চাপলে পুরো `formData` একবারে backend-এ যায়।

```jsx
// frontend/src/pages/Onboarding/OnboardingFlow.jsx
const handleComplete = async () => {
    setIsLoading(true);
    const uid = user?.uid || 'demo-user';
    const response = await fetch(`${API_BASE}/api/onboarding`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ uid, ...formData })
    });
    const data = await response.json();
    if (data.success) { toast.success('Onboarding complete! 🚀'); setIsCompleted(true); }
    // ...
};
```

### ধাপ ৫ — Backend: skill-key normalize + দুই collection-এ লেখা

Backend-এর সবচেয়ে দরকারি অংশ — skill নাম গুলোকে DB-key হিসেবে ব্যবহারযোগ্য বানানো (`Node.js` → `nodejs`, `Version Control` → `version-control`), তারপর `User` আর `Progress` দুটোতেই লেখা।

```js
// backend/routes/onboarding.routes.js
const cleanedSkills = (skills || []).map(skill => ({
    original: skill,
    key: skill.toLowerCase()
        .replace(/\./g, '')          // Node.js → nodejs
        .replace(/\s+/g, '-')        // "Version Control" → version-control
        .replace(/[^a-z0-9-]/g, '')  // বাকি special char বাদ
}));

await User.findOneAndUpdate({ uid: String(uid) }, {
    'onboarding.role': role, 'onboarding.skills': skills,
    'onboarding.targetCareer': targetCareer, 'onboarding.completed': true,
    skillStrength: cleanedSkills.reduce((acc, s) => { acc[s.key] = 40; return acc; }, {})
}, { new: true, upsert: true });

// Progress-এও initial data — dashboard/radar সাথে সাথে ভরে যায়
await Progress.findOneAndUpdate({ uid: String(uid) }, { $set: {
    targetRole: targetCareer || "Full Stack Developer",
    skillStrength: cleanedSkills.reduce((acc, s) => { acc[s.key] = 40; return acc; }, {}),
    milestones: [{ title: "Onboarding Complete", status: "completed", date: new Date().toISOString().split('T')[0] }],
    percentage: 10
}}, { upsert: true, new: true });
```

প্রতিটা skill-এ শুরুতে `40` value দেওয়া হয় — মানে radar-এ user একটা baseline নিয়ে শুরু করে, খালি না।

### ধাপ ৬ — সফল হলে PDF download

`isCompleted` true হলে success screen দেখায়, যেখানে `PDFDownloadLink` দিয়ে formData থেকেই Roadmap ও Resume PDF বানানো যায় — সব client-side, কোনো server লাগে না।

```jsx
// frontend/src/pages/Onboarding/OnboardingFlow.jsx
<PDFDownloadLink document={<RoadmapPDF roadmapData={formData} userName={user?.displayName||'Voyager'}/>}
    fileName={`SkillVoyager_Roadmap_${new Date().toISOString().split('T')[0]}.pdf`}>
    {({ loading }) => <button disabled={loading}>Download Roadmap PDF</button>}
</PDFDownloadLink>
```

## মূল ধারণা (Key Concepts)

- **Multi-step wizard** — বড় form-কে ছোট ধাপে ভেঙে completion rate বাড়ানো।
- **Lifted state** — সব step-এর data parent-এ (`formData`) রাখা, step গুলো stateless।
- **Controlled inputs** — প্রতিটা input-এর মান React state থেকে আসে, `updateFormData` দিয়ে বদলায়।
- **Per-step validation** — অসম্পূর্ণ ধাপে আটকে দেওয়া, শেষে একবারে না দেখিয়ে।
- **Key-based remount for animation** — `key={currentStep}` দিয়ে exit/enter animation trigger।
- **Data normalization** — skill নাম-কে consistent DB key-তে রূপান্তর।
- **Dual-write** — একই data দুই collection-এ (User + Progress) লেখা, downstream feature-কে ready রাখতে।
- **Client-side PDF generation** — server ছাড়াই browser-এ document বানানো।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. Multi-step করলেন কেন, এক page-এর form না বানিয়ে? এর মাপযোগ্য সুবিধা কী?
**উত্তর:** UX ও conversion-এর জন্য। একটা লম্বা form (৫টা প্রশ্ন একসাথে) নতুন user-কে overwhelm করে, drop-off বাড়ে। ধাপে ভাগ করলে প্রতি screen-এ একটা ছোট, হজমযোগ্য কাজ থাকে — user কম চাপ অনুভব করে, progress indicator "প্রায় শেষ" অনুভূতি দেয়, ফলে completion rate বাড়ে। এটা একটা প্রমাণিত UX pattern (progressive disclosure)। Trade-off — বেশি click লাগে এবং state management একটু জটিল; কিন্তু onboarding-এর মতো high-stakes মুহূর্তে completion-ই আসল, তাই এই trade-off যুক্তিসঙ্গত।

### ২. State সব parent-এ রাখলেন কেন, প্রতিটা step-এ নিজের state না রেখে?
**উত্তর:** কারণ data-টা step-এর চেয়ে বেশি দিন বাঁচতে হবে। User ধাপ ২ থেকে ৩-এ গিয়ে আবার ২-এ ফিরলে, ধাপ ২-এর উত্তর মনে থাকা দরকার। যদি প্রতিটা step-এ local state রাখতাম, ধাপ বদলে component unmount হলে সেই state হারিয়ে যেত। State-কে parent-এ "lift" করায় formData সব ধাপ জুড়ে টিকে থাকে, আর submit-এর সময় পুরো data এক জায়গায় থাকে। এতে step component গুলো "dumb"/reusable থাকে — শুধু props নেয়, নিজে কিছু মনে রাখে না।

### ৩. `key={currentStep}` না দিলে animation-এ কী হতো?
**উত্তর:** React reconciliation একই position-এর `motion.div`-কে একই element মনে করে শুধু ভেতরের content বদলে দিত — নতুন element mount/unmount হতো না, ফলে `AnimatePresence`-এর exit/enter animation trigger-ই হতো না, ধাপ হঠাৎ বদলে যেত animation ছাড়া। `key={currentStep}` বদলানোয় React পুরনো div-কে "আলাদা element" গণ্য করে unmount (exit animation) করে ও নতুন mount (enter animation) করে। মানে key এখানে animation-এর trigger। এটা একটা সাধারণ কিন্তু গুরুত্বপূর্ণ React সূক্ষ্মতা।

### ৪. Onboarding data দুই জায়গায় (User + Progress) লেখা হচ্ছে — এটা কি data duplication না? ঝুঁকি কী?
**উত্তর:** হ্যাঁ, এটা intentional denormalization, এবং ঝুঁকিও আছে। কারণ — `Progress` collection dashboard/radar দ্রুত পড়ার জন্য আলাদা রাখা, প্রতিবার User doc-এর ভারী nested object না খুঁজে। সুবিধা: read fast, feature গুলো decoupled। ঝুঁকি: দুই জায়গার data **out of sync** হতে পারে — এক জায়গায় skillStrength বদলালে অন্যটা পুরনো থেকে যেতে পারে (consistency সমস্যা)। এখানে দুটো `findOneAndUpdate` আলাদা call, তাই একটা সফল অন্যটা fail করলে partial state-ও হতে পারে (transaction নেই)। উন্নত করতে হলে হয় single source of truth রাখতাম, নাহলে MongoDB transaction দিয়ে দুটো write atomic করতাম।

### ৫. Skill নাম normalize (`Node.js` → `nodejs`) করার দরকার হলো কেন?
**উত্তর:** কারণ এই skill নামগুলো MongoDB-তে **object/Map-এর key** হিসেবে ব্যবহৃত হয় (`skillStrength`)। MongoDB field name-এ `.` special (nested path বোঝায়) এবং space/special char সমস্যা করে — `Node.js` কে key করলে Mongo সেটাকে `Node` object-এর `js` field ভাবত। তাই `.` বাদ, space-কে `-`, বাকি special char বাদ দিয়ে একটা safe, consistent key বানানো হয়। এতে পরে quiz/milestone থেকে skill boost করার সময় একই normalize করলে key মিলবে (যেমন quiz controller-এও `react` key তৈরি হয়)। Normalize না করলে একই skill ভিন্ন জায়গায় ভিন্ন key পেয়ে radar-এ duplicate/mismatch হতো।

### ৬. প্রতিটা skill-এ শুরুতে `40` কেন, `0` না?
**উত্তর:** এটা UX-driven সিদ্ধান্ত। সব skill `0` থেকে শুরু করলে radar chart প্রায় একটা বিন্দুতে চুপসে যেত — user-এর মনে হতো "আমি কিছুই জানি না", demotivating। `40` একটা baseline দেয়, radar-টা ভরা দেখায়, আর "target 80%"-এর দিকে যাওয়ার একটা visible gap থাকে — যা motivating। এটা technically arbitrary, কিন্তু gamification-এ starting point ইতিবাচক রাখা গুরুত্বপূর্ণ। দুর্বলতা — এটা user-এর আসল দক্ষতা প্রতিফলিত করে না; আদর্শভাবে একটা assessment quiz দিয়ে baseline নির্ধারণ করা উচিত ছিল।

### ৭. Submit-এর সময় `user?.uid || 'demo-user'` — এই fallback কেন, এতে সমস্যা কী?
**উত্তর:** Fallback রাখা হয়েছে যাতে auth এখনো load না হলেও বা কোনো কারণে uid না থাকলেও flow না ভাঙে (demo/development সুবিধা)। কিন্তু এতে সমস্যা আছে — একাধিক user যদি uid ছাড়া submit করে, সবাই `'demo-user'` key-তে লিখবে, ফলে একে অন্যের onboarding data overwrite করবে (upsert একই doc-এ)। Production-এ এটা bug: uid না থাকলে submit block করা বা login-এ পাঠানো উচিত, একটা shared fake id-তে লেখা নয়। যেহেতু route-টা `PrivateRoute`-এর ভেতরে, বাস্তবে uid থাকার কথা — তবু defensive fallback-টা এখানে ভুল ডিফল্ট বেছেছে।

### ৮. PDF generation client-side করলেন কেন, server-side না?
**উত্তর:** কারণ PDF-এর data (formData) ইতিমধ্যে client-এ আছে এবং এটা user-specific, one-off document। `@react-pdf/renderer` দিয়ে browser-এ render করলে — (ক) server-এ কোনো load পড়ে না, (খ) কোনো round-trip/latency নেই, (গ) server-এ headless browser/PDF engine host করার infra লাগে না। Trade-off: PDF generation ভারী library (bundle বড় করে) এবং পুরোটা user-এর device-এ চলে, তাই দুর্বল device-এ ধীর হতে পারে; আর server-side হলে একটা canonical, verifiable copy রাখা যেত (যেমন certificate)। এই use case-এ (ব্যক্তিগত roadmap/resume) client-side-ই সঠিক পছন্দ।

### ৯. এই wizard-এ user browser refresh করলে কী হয়? কীভাবে উন্নত করতেন?
**উত্তর:** এখন refresh করলে সব হারিয়ে যায় — `formData` React state-এ, memory-তে; refresh মানে state reset, user আবার ধাপ ১ থেকে শুরু। onboarding-এর মতো multi-step flow-এ এটা হতাশাজনক। উন্নত করতে প্রতি ধাপে `formData`-কে `localStorage`/`sessionStorage`-এ persist করতাম এবং mount-এ সেখান থেকে restore করতাম, বা backend-এ draft হিসেবে ধাপে ধাপে save করতাম। তাহলে refresh/দুর্ঘটনাবশত tab বন্ধ হলেও user যেখানে ছিল সেখান থেকে চালিয়ে যেতে পারত। এটা completion rate আরও বাড়াত।

### ১০. Step component গুলো "dumb" (props-only) রাখলেন — এর architectural সুবিধা কী?
**উত্তর:** এটা **separation of concerns** ও **reusability** দেয়। Step গুলো শুধু `formData` পড়ে আর `updateFormData` কল করে — তারা জানে না কীভাবে submit হয়, কোন ধাপ পরে আসে, বা validation কীভাবে হয়। ফলে (ক) এদের test করা সহজ (pure input/output), (খ) ধাপের ক্রম বদলানো বা নতুন ধাপ যোগ করা orchestrator-এ সীমাবদ্ধ থাকে, step-এ হাত দিতে হয় না, (গ) একই step অন্য context-এ reuse করা যায়। Business logic (state, navigation, API) থাকে এক জায়গায় (`OnboardingFlow`), presentation থাকে আলাদা — এটাই container/presentational pattern, যা বড় হলে maintainable রাখে।
