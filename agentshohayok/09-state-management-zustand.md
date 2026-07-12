# ০৯ — State Management (Zustand + Persist)

## Feature-টা আসলে কী?

App-এর অনেক তথ্য একাধিক component-এ লাগে — user login করা কিনা, তার phone, role, ভাষা, theme ইত্যাদি। এই shared state manage করতে **Zustand** নামের একটা হালকা state-management library ব্যবহার করা হয়েছে। আর `persist` middleware দিয়ে কিছু state browser-এর localStorage-এ রাখা হয়, যাতে refresh-এও টিকে থাকে।

## কোন কোন file জড়িত?

| Store | File | কী রাখে |
|-------|------|---------|
| Auth | `src/lib/store/auth.ts` | phone, pin, authed, role, session expiry |
| Profile | `src/lib/store/profile.ts` | নাম, দোকান, avatar |
| Theme | `src/lib/store/theme.ts` | light/dark/system |
| Language | `src/lib/i18n/index.ts` | bn/en |
| Recharge PIN | `src/lib/store/recharge-pin.ts` | recharge-এর সময় extra PIN |

### জড়িত file গুলো সংক্ষেপে

- **`src/lib/store/auth.ts`** — সবচেয়ে গুরুত্বপূর্ণ store: phone, pin, authed, role, session expiry; login/logout/lock, session validity check, আর persist সহ role re-derivation।
- **`src/lib/store/profile.ts`** — user-এর নাম, দোকানের নাম, avatar ইত্যাদি profile তথ্য।
- **`src/lib/store/theme.ts`** — light/dark/system theme পছন্দ ও তা DOM-এ apply করা।
- **`src/lib/store/recharge-pin.ts`** — recharge-এর সময় দরকার হওয়া extra PIN-এর state (session-ভিত্তিক)।
- **`src/lib/i18n/index.ts → useLangStore`** — ভাষার state (bn/en), persist সহ (i18n doc-এ বিস্তারিত)।

## এটা কীভাবে কাজ করে

### ধাপ ১ — একটা store বানানো

Zustand-এ store মানে একটা hook। ভেতরে state ও সেই state বদলানোর function থাকে:

```ts
// src/lib/store/auth.ts (সরলীকৃত)
export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      phone: null,
      authed: false,
      role: "agent",
      login: (pin) => set({ authed: true, role: roleForPhone(get().phone), ... }),
      logout: () => { /* server session-ও মুছে */ set({ authed: false, ... }); },
    }),
    { name: "rp.auth" },   // localStorage key
  ),
);
```

### ধাপ ২ — Component-এ ব্যবহার

```tsx
const { authed, role } = useAuthStore();          // পুরো state
const lang = useLangStore((s) => s.lang);         // শুধু একটা field (selector)
```

Selector (`(s) => s.lang`) ব্যবহার করলে component শুধু ঐ নির্দিষ্ট field বদলালেই re-render হয় — অপ্রয়োজনীয় re-render কমে, performance ভালো হয়।

### ধাপ ৩ — Persist ও Hydration

`persist` state-টা localStorage-এ রাখে। কিন্তু এখানে একটা সূক্ষ্ম সমস্যা আছে — **hydration**। Server-এ (SSR) localStorage নেই, তাই প্রথম render-এ state থাকে default (logged-out); তারপর browser-এ localStorage থেকে আসল state "hydrate" হয়। এই দুই render যেন না গোলমাল করে, সেজন্য code hydration সম্পূর্ণ হওয়ার জন্য অপেক্ষা করে:

```ts
// AuthGuard-এ
useEffect(() => {
  const unsub = useAuthStore.persist.onFinishHydration(() => setReady(true));
  setReady(useAuthStore.persist.hasHydrated());
  return unsub;
}, []);
```

### ধাপ ৪ — Security: persisted state কখনো অন্ধভাবে বিশ্বাস নয়

localStorage user নিজে edit করতে পারে। তাই কেউ যদি localStorage-এ `role: "admin"` বসিয়ে দেয়, সেটা যেন কাজ না করে — role সবসময় phone থেকে **আবার derive** করা হয়:

```ts
onRehydrateStorage: () => (state) => {
  if (state) state.role = roleForPhone(state.phone);   // stale/জাল role ঠিক করা
},
```

মনে রাখতে হবে — এই client-side role শুধু UI-এর জন্য; আসল authorization তো server-এ database session দিয়েই হয়।

## গুরুত্বপূর্ণ concept গুলো

- **Global vs local state**: অনেক component-এ লাগে এমন state store-এ; একটা component-এর নিজের state `useState`-এ।
- **Selector**: store থেকে শুধু দরকারি অংশ নেওয়া → কম re-render।
- **Hydration**: SSR-এর server render আর client render মেলানো।
- **Client state ≠ security boundary**: client state UI-এর সুবিধার জন্য, নিরাপত্তার ভিত্তি নয়।

---

## Interview Questions

### ১. Zustand কেন, React-এর নিজের Context API বা Redux নয়?

**উত্তর:** তিনটাই global state manage করে, কিন্তু trade-off আলাদা। **Context API** ছোট state-এ ভালো, তবে context-এর value বদলালে তার নিচের সব consumer re-render হয় (selector নেই), আর অনেক আলাদা state-এ অনেক provider nesting তৈরি হয়। **Redux** শক্তিশালী কিন্তু boilerplate বেশি (action, reducer, dispatch)। **Zustand** এই মাঝামাঝি: খুব কম boilerplate (শুধু একটা hook), built-in selector দিয়ে fine-grained re-render নিয়ন্ত্রণ, provider wrapping লাগে না, আর `persist`-এর মতো middleware সহজে যোগ করা যায়। এই project-এর মতো মাঝারি app-এ Zustand দ্রুত ও পরিচ্ছন্ন — তাই বেছে নেওয়া।

### ২. Zustand-এ selector (`(s) => s.lang`) ব্যবহার করলে কী সুবিধা?

**উত্তর:** Selector ছাড়া `useAuthStore()` করলে component পুরো store subscribe করে — store-এর যেকোনো field বদলালেই সেটা re-render হয়, এমনকি সে যে field ব্যবহার করে না সেটা বদলালেও। Selector (`useLangStore((s) => s.lang)`) দিয়ে component বলে দেয় "আমি শুধু `lang`-এ আগ্রহী", তাই কেবল `lang` বদলালেই সে re-render হয়। বড় app-এ এটা অপ্রয়োজনীয় re-render অনেক কমিয়ে performance বাড়ায়। নিয়ম: store থেকে যতটুকু দরকার ঠিক ততটুকুই select করা।

### ৩. "Hydration" সমস্যা কী, আর এটা ঠিকমতো handle না করলে কী হয়?

**উত্তর:** Next.js প্রথমে server-এ HTML render করে (SSR), যেখানে localStorage নেই — তাই persisted state তখন default (যেমন logged-out) থাকে। তারপর browser-এ JS load হয়ে localStorage থেকে আসল state "hydrate" করে। যদি server-render আর client-এর প্রথম render আলাদা হয় (server বলল logged-out, client বলল logged-in), React একটা **hydration mismatch** error দেয় এবং UI ঝলকাতে (flash) পারে। এটা ঠেকাতে code `hasHydrated()`/`onFinishHydration` দিয়ে অপেক্ষা করে — hydration শেষ না হওয়া পর্যন্ত একটা skeleton/loading দেখায়, তারপর আসল state অনুযায়ী UI render করে। ফলে server ও client-এর প্রথম render মিলে যায়, mismatch হয় না।

### ৪. persisted state (localStorage)-কে কেন "অবিশ্বাস্য" ধরা হয়, আর code কীভাবে রক্ষা করে?

**উত্তর:** localStorage সম্পূর্ণ client-এর নিয়ন্ত্রণে — user browser dev-tools খুলে যেকোনো মান বদলাতে পারে, যেমন `role: "agent"`-কে `role: "admin"` বানিয়ে দেওয়া। তাই persisted state অন্ধভাবে বিশ্বাস করলে জাল role দিয়ে UI-তে admin সেজে বসা যেত। Code এটা ঠেকায় `onRehydrateStorage`-এ — state load হওয়ার সময় role আবার phone থেকে derive করে (`roleForPhone`), তাই localStorage-এ যা-ই থাকুক, সঠিক role বসে। তবে মূল প্রতিরক্ষা হলো — এই client-side role শুধু UI দেখানোর জন্য; আসল admin API access তো server database session দিয়ে যাচাই হয়, যা client বদলাতে পারে না।

### ৫. `logout`-এ শুধু client state clear না করে server-এও `/api/auth/logout` call করা হয় কেন?

**উত্তর:** শুধু client state (zustand/localStorage) clear করলে browser-এ UI logged-out দেখাবে ঠিকই, কিন্তু server-এর database-এ Session row আর httpOnly cookie **টিকে থাকবে** — অর্থাৎ session আসলে এখনো valid, কেউ সেই cookie নিয়ে API access করতে পারবে। প্রকৃত logout মানে session-টাকে server-এও অকার্যকর করা: `/api/auth/logout` DB থেকে Session row মুছে দেয় ও cookie clear করে। এটাই stateful session-এর সুবিধা — instant revocation। তাই client + server দুই দিকেই logout সম্পন্ন করা জরুরি, নাহলে "logout" আসলে অসম্পূর্ণ থাকে।

### ৬. Zustand-এ `create<AuthState>()( ... )` — এই double-call/curry syntax কেন?

**উত্তর:** এটা মূলত TypeScript-এর জন্য। `create<AuthState>()(...)` — প্রথম বন্ধনীতে টাইপ argument দিয়ে দ্বিতীয় বন্ধনীতে actual store definition দেওয়া হয়। এই curried form দরকার কারণ `persist`-এর মতো middleware ব্যবহার করলে TypeScript সরাসরি `create<AuthState>(...)` থেকে টাইপ ঠিকমতো infer করতে পারে না; middleware-এর ভেতর দিয়ে টাইপ pass করতে গেলে এই দুই-ধাপের pattern লাগে। JavaScript-এ এটা লাগত না, কিন্তু type-safe রাখতে Zustand-এর ডকেই এই syntax সুপারিশ করা হয়। সংক্ষেপে — middleware + সঠিক টাইপ inference একসাথে পেতে এই curry।

### ৭. Store-এর ভেতরে `get()` function-টা কখন লাগে?

**উত্তর:** `set` দিয়ে state বদলানো যায়, আর `get()` দিয়ে **বর্তমান state পড়া** যায় — বিশেষত যখন একটা action-এর ভেতরে অন্য field-এর current মান দরকার। যেমন `auth.ts`-এর `login`-এ role নির্ধারণ করতে current phone লাগে: `role: roleForPhone(get().phone)`। এখানে `get()` ছাড়া বর্তমান phone জানার উপায় থাকত না (action-এর argument-এ তো শুধু নতুন pin আসছে)। আরেকটা উদাহরণ — `isSessionValid()` যেখানে `get().authed` ও `get().sessionExpiresAt` পড়ে সিদ্ধান্ত নেয়। মোটকথা — action-এর ভেতরে "আগে state কী ছিল তার ওপর নির্ভর করে নতুন state" ঠিক করতে `get()` লাগে।

### ৮. persist-এ কোন field save হবে তা `partialize` দিয়ে বেছে নেওয়া যায় — PIN persist করা কি নিরাপদ?

**উত্তর:** এটা একটা গুরুত্বপূর্ণ নিরাপত্তা বিবেচনা। `persist`-এ default-ভাবে পুরো state localStorage-এ যায়, কিন্তু `partialize` option দিয়ে বলা যায় কোন field গুলো শুধু save হবে। PIN-এর মতো সংবেদনশীল জিনিস plaintext-এ localStorage-এ রাখা **ঝুঁকিপূর্ণ** — কারণ যেকোনো XSS script বা device-এ access পাওয়া কেউ সেটা পড়তে পারে। আদর্শভাবে PIN কখনো persist করা উচিত নয়; দরকার হলে শুধু একটা non-sensitive flag (যেমন "pin set আছে কিনা") রাখা যায়, আসল PIN নয়। এই project-এ যেহেতু আসল authentication এখন server-side session cookie দিয়ে হয়, client-এর PIN state মূলত UX-এর জন্য — তবু best practice হলো sensitive field গুলো `partialize` দিয়ে persist থেকে বাদ দেওয়া।

### ৯. একাধিক browser tab খোলা থাকলে, এক tab-এ logout করলে অন্য tab-এ কী হয়?

**উত্তর:** Default-ভাবে Zustand-এর in-memory state প্রতি tab-এ আলাদা, তাই এক tab-এ logout করলে অন্য tab-এর React state সাথে সাথে বদলায় না — সেই tab UI-তে এখনো logged-in দেখাতে পারে। তবে দুটো জিনিস এটাকে প্রভাবিত করে: (ক) `persist` localStorage ব্যবহার করে, আর localStorage-এ পরিবর্তন হলে browser একটা `storage` event ছোড়ে যা অন্য tab শুনে state sync করতে পারে (Zustand persist-এ এটা configure করা যায়); (খ) যেহেতু আসল auth server-side session cookie, logout API session মুছে দেয় — তাই অন্য tab পরের কোনো API call করলে 401 পেয়ে বুঝবে session শেষ। নিখুঁত UX চাইলে cross-tab sync (storage event শোনা) যোগ করা যায়, যাতে এক tab-এ logout করলে সব tab সাথে সাথে logged-out হয়।

### ১০. একটা Zustand store test করা কীভাবে যায়?

**উত্তর:** Zustand-এর একটা বড় সুবিধা — store আসলে একটা সাধারণ function/hook, React ছাড়াই এর logic test করা যায়। উপায়: (ক) store থেকে state ও action সরাসরি নিয়ে (`useAuthStore.getState()`) action call করে তারপর state যাচাই করা — যেমন `login("12345")` করার পর `getState().authed === true` কিনা দেখা; (খ) প্রতি test-এর আগে store একটা পরিচিত initial state-এ reset করা, যাতে test গুলো একে অন্যকে প্রভাবিত না করে; (গ) persist middleware থাকলে test-এ localStorage mock করা বা persist বন্ধ রাখা। যেহেতু business logic (যেমন `roleForPhone`, `isSessionValid`) pure function-এর কাছাকাছি, এগুলো unit test করা সহজ ও দ্রুত — UI render না করেই state management-এর সঠিকতা যাচাই করা যায়।
