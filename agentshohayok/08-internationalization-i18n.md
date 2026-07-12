# ০৮ — Internationalization (i18n) — বাংলা/English

## Feature-টা আসলে কী?

App-টা দুই ভাষায় চলে — **বাংলা** ও **English**। User একটা toggle দিয়ে ভাষা বদলাতে পারে, আর পুরো UI (text, সংখ্যা, টাকার অঙ্ক) সাথে সাথে সেই ভাষায় দেখায়। এই বহুভাষিক support-কে বলে **i18n (internationalization)**।

Bangladesh-এর agent-দের অনেকেই English-এ স্বচ্ছন্দ নন, তাই বাংলা support ব্যবহারযোগ্যতার জন্য অত্যন্ত গুরুত্বপূর্ণ।

## কোন কোন file জড়িত?

| Layer | File | কাজ |
|-------|------|-----|
| Store | `src/lib/i18n/index.ts` | `useLangStore`, `useT`, number/currency format |
| Translations | `src/lib/i18n/translations.ts` | সব text-এর বাংলা/English map |
| ব্যবহার | সব component (`useT()`, `useLangStore()`) | UI-তে অনূদিত text দেখায় |

### জড়িত file গুলো সংক্ষেপে

- **`src/lib/i18n/index.ts`** — i18n-এর মূল: `useLangStore` (কোন ভাষা সক্রিয়, persist সহ), `useT` hook (key → অনূদিত text), আর `formatNum`/`formatCurrency` (সংখ্যা/টাকা localize)।
- **`src/lib/i18n/translations.ts`** — সব UI text-এর একটা বড় map, প্রতিটা key-র নিচে `{ bn, en }` মান; এখানেই সব অনুবাদ কেন্দ্রীভূত।
- **সব component (যারা `useT()`/`useLangStore()` ব্যবহার করে)** — text সরাসরি না লিখে `t("key")` দিয়ে দেখায়, তাই ভাষা বদলালে নিজে থেকেই re-render হয়ে নতুন ভাষায় আসে।

## এটা কীভাবে কাজ করে

### ধাপ ১ — ভাষার state (zustand + persist)

কোন ভাষা এখন সক্রিয় সেটা একটা zustand store-এ থাকে, আর `persist` দিয়ে localStorage-এ save হয় — তাই page refresh করলেও ভাষা মনে থাকে:

```ts
// src/lib/i18n/index.ts
export const useLangStore = create<LangState>()(
  persist(
    (set) => ({
      lang: "bn",                              // default বাংলা
      setLang: (lang) => set({ lang }),
    }),
    { name: "rp.lang" },
  ),
);
```

### ধাপ ২ — Translation map (key → দুই ভাষার text)

প্রতিটা text একটা key দিয়ে চিহ্নিত, আর দুই ভাষায় তার মান রাখা:

```ts
// src/lib/i18n/translations.ts
export const translations = {
  auth_logout: { bn: "লগ আউট", en: "Log out" },
  nav_dashboard: { bn: "ড্যাশবোর্ড", en: "Dashboard" },
  // ...
};
```

### ধাপ ৩ — `useT` hook দিয়ে অনুবাদ পাওয়া

Component-এ `useT()` hook দিয়ে একটা function পাওয়া যায় যেটা key দিলে current ভাষার text ফেরত দেয়:

```ts
export function useT() {
  const lang = useLangStore((s) => s.lang);
  return useCallback((key: TKey) => translations[key]?.[lang] ?? key, [lang]);
}

// component-এ:
const t = useT();
<span>{t("nav_dashboard")}</span>   // বাংলা হলে "ড্যাশবোর্ড", নাহলে "Dashboard"
```

ভাষা বদলালে `lang` state বদলায়, তাই যেসব component `useT`/`useLangStore` ব্যবহার করে সেগুলো নিজে থেকেই re-render হয়ে নতুন ভাষায় দেখায়।

### ধাপ ৪ — সংখ্যা ও টাকা localize করা

শুধু text নয়, সংখ্যাও বাংলায় দেখানো হয় (`0-9` → `০-৯`):

```ts
const bnDigits = ["০","১","২","৩","৪","৫","৬","৭","৮","৯"];

export function formatNum(n, lang) {
  const s = typeof n === "number" ? n.toLocaleString("en-US") : String(n);
  return lang === "en" ? s : s.replace(/\d/g, (d) => bnDigits[Number(d)]);
}

export function formatCurrency(n, lang) {
  const formatted = formatNum(Math.round(n), lang);
  return lang === "bn" ? `৳ ${formatted}` : `৳${formatted}`;
}
```

## গুরুত্বপূর্ণ concept গুলো

- **Key-based translation**: text সরাসরি না লিখে key দিয়ে referencing — একই key দুই ভাষায় map করা।
- **Reactive language switch**: ভাষা state বদলালে UI নিজে থেকেই update হয় (React re-render)।
- **Locale-aware formatting**: শুধু শব্দ নয়, সংখ্যা/মুদ্রার format-ও ভাষা অনুযায়ী।
- **Persistence**: ভাষার পছন্দ localStorage-এ থাকে।

---

## Interview Questions

### ১. Text সরাসরি component-এ না লিখে key-based translation (`t("nav_dashboard")`) ব্যবহার করার সুবিধা কী?

**উত্তর:** সরাসরি text লিখলে দুই ভাষা support করতে প্রতিটা জায়গায় `if (lang === "bn") ... else ...` লিখতে হতো, যা repetitive ও ভুলপ্রবণ। Key-based approach-এ প্রতিটা text একটা key দিয়ে referenced, আর সব অনুবাদ এক জায়গায় (`translations.ts`)। সুবিধা: (ক) নতুন ভাষা যোগ করা সহজ — শুধু map-এ আরেকটা language key যোগ; (খ) সব text এক জায়গায় থাকায় manage/proofread করা সহজ; (গ) একই text অনেক জায়গায় ব্যবহার হলে একবার বদলালেই সব জায়গায় বদলায়; (ঘ) কোনো key-এর অনুবাদ না থাকলে fallback হিসেবে key নিজেই দেখায়, app crash করে না।

### ২. ভাষা toggle করলে পুরো UI সাথে সাথে বদলায় কীভাবে?

**উত্তর:** ভাষাটা একটা zustand store-এ (`useLangStore`) state হিসেবে থাকে। `useT` এবং যেসব component ভাষা দেখায়, তারা এই store subscribe করে (`useLangStore((s) => s.lang)`)। `setLang` দিয়ে ভাষা বদলালে store-এর state বদলায়, আর React-এর reactivity অনুযায়ী ঐ state-এ subscribe করা প্রতিটা component **নিজে থেকে re-render** হয়ে নতুন ভাষার text দেখায়। অর্থাৎ কোনো manual DOM update বা page reload লাগে না — state পরিবর্তন → re-render → নতুন UI, এটাই React-এর declarative model-এর সৌন্দর্য।

### ৩. শুধু শব্দ অনুবাদ যথেষ্ট নয় কেন, সংখ্যাও কেন localize করা হয়?

**উত্তর:** সত্যিকারের localization মানে ব্যবহারকারীর কাছে স্বাভাবিক মনে হওয়া — আর বাংলাভাষী user-এর কাছে `৳ ১,৫০০` অনেক বেশি পরিচিত ও পাঠযোগ্য `৳1,500`-এর চেয়ে। তাই `formatNum` ইংরেজি digit (`0-9`)-কে বাংলা digit (`০-৯`)-এ রূপান্তর করে, আর `formatCurrency` মুদ্রা চিহ্ন ও spacing ঠিক করে। বড় পরিসরে locale-aware formatting-এ তারিখ, সময়, দশমিক আলাদা করার চিহ্ন — সবই ভাষা/অঞ্চল অনুযায়ী বদলায়। শুধু শব্দ অনুবাদ করলে অর্ধেক কাজ হয়; সংখ্যা/মুদ্রা/তারিখ localize না করলে অভিজ্ঞতা খাপছাড়া লাগে।

### ৪. ভাষার পছন্দ `persist` দিয়ে localStorage-এ রাখা হয় কেন?

**উত্তর:** যদি persist না থাকত, তাহলে প্রতিবার page refresh বা app পুনরায় খোলার সময় ভাষা default (বাংলা)-তে ফিরে যেত, এবং English-পছন্দকারী user-কে বারবার toggle করতে হতো — বাজে অভিজ্ঞতা। `persist` middleware store-এর state localStorage-এ (`rp.lang` key-তে) সংরক্ষণ করে, তাই user-এর পছন্দ device-এ মনে থাকে এবং পরের বার সেই ভাষাতেই app খোলে। এটা একটা user preference, তাই এটা মনে রাখা ব্যবহারযোগ্যতার জন্য জরুরি।

### ৫. এই সরল i18n approach-এর সীমাবদ্ধতা কী, বড় app-এ কী ভিন্নভাবে করা হয়?

**উত্তর:** এই approach ছোট-মাঝারি app-এর জন্য চমৎকার, কিন্তু কিছু সীমাবদ্ধতা আছে: (ক) সব translation bundle-এর সাথে load হয়, ভাষা বেশি/text বেশি হলে bundle বড় হয় (বড় app-এ ভাষা অনুযায়ী lazy-load করা হয়); (খ) **pluralization** (১টা item vs ২টা items) ও **interpolation** (`"{count} টি নতুন বার্তা"`)-এর built-in support নেই; (গ) gender/context-নির্ভর অনুবাদ handle করা কঠিন। বড় app-এ সাধারণত `next-intl`, `react-i18next`-এর মতো প্রতিষ্ঠিত library ব্যবহার হয়, যেগুলো plural rules, variable interpolation, ভাষা-ভিত্তিক code splitting, এমনকি server-side rendering-এর সাথে integration দেয়। তবে এই project-এর scale-এ custom সমাধানই যথেষ্ট ও পরিষ্কার।

### ৬. `TKey` type দিয়ে translation key type-safe করা হয়েছে — ভুল key দিলে কী হয়?

**উত্তর:** `useT`-এর function-টা `(key: TKey) => ...` — এখানে `TKey` হলো `translations` object-এর সব key-এর union type (TypeScript-এ `keyof typeof translations`-এর মতো)। ফলে কেউ যদি এমন একটা key লেখে যা translations-এ নেই (যেমন `t("nav_dashbord")` — বানান ভুল), তাহলে **compile-time-এই** TypeScript error দেবে, code চালানোর আগেই ধরা পড়বে। এটা একটা বড় সুবিধা — runtime-এ "key পাওয়া গেল না" নীরব bug হওয়ার বদলে editor-এই লাল দাগ দেখা যায়। আর নিরাপত্তা জাল হিসেবে runtime-এ `translations[key]?.[lang] ?? key` — key না মিললে অন্তত key-টাই দেখায়, app crash করে না।

### ৭. Default ভাষা `"bn"` — SSR-এর সময় এটা hydration mismatch তৈরি করে না?

**উত্তর:** সাবধানে handle না করলে করতে পারত। Server-এ persist store-এর default (`"bn"`) দিয়ে render হয়, আর client-এ localStorage থেকে যদি অন্য ভাষা (`"en"`) hydrate হয়, তাহলে server-render আর প্রথম client-render আলাদা text দেখাবে — React hydration mismatch। এই ধরনের সমস্যা এড়াতে সাধারণ কৌশল — hydration সম্পূর্ণ হওয়া পর্যন্ত অপেক্ষা করা, বা ভাষা-নির্ভর অংশ hydration-এর পরে render করা (এই project-এর guard/landing-এ hydration gating-এর মতো)। মূল শিক্ষা — যেকোনো persisted/client-only state (ভাষা, theme) SSR-এর সাথে ব্যবহার করলে server ও প্রথম client render যেন মেলে, সেটা নিশ্চিত করতে হয়, নাহলে ঝলকানি বা console error আসে।

### ৮. নতুন একটা ভাষা (যেমন হিন্দি) যোগ করতে হলে কী কী বদলাতে হবে?

**উত্তর:** এই design-এ ধাপগুলো মোটামুটি: (ক) `Lang` type-এ নতুন মান যোগ (`"bn" | "en" | "hi"`); (খ) `translations.ts`-এর প্রতিটা key-তে `hi` অনুবাদ যোগ করা — এটাই সবচেয়ে বড় কাজ, কারণ প্রতিটা string অনুবাদ করতে হবে; (গ) `formatNum`-এ হিন্দি digit map যোগ করা যদি সংখ্যাও localize করতে হয়; (ঘ) ভাষা toggle UI-তে নতুন option। যেহেতু সব text key-based ও এক জায়গায়, structure বদলাতে হয় না — শুধু data বাড়াতে হয়। এটাই key-based i18n-এর সুবিধা: নতুন ভাষা যোগ করা মানে মূলত অনুবাদ পূরণ করা, code পরিবর্তন নয়। বড় হলে অনুবাদ missing আছে কিনা তা ধরার জন্য type-safety ও একটা lint/script সাহায্য করে।

### ৯. কিছু page-এ সরাসরি `lang === "bn" ? "..." : "..."` লেখা, আবার কোথাও `t()` — এই mixed approach-এ সমস্যা কী?

**উত্তর:** এটা একটা inconsistency এবং সাধারণত `t()`-ই ভালো practice। সরাসরি `lang === "bn" ? ... : ...` লেখার সমস্যা: (ক) text component-এর ভেতরে ছড়িয়ে থাকে, এক জায়গা থেকে সব অনুবাদ manage/proofread করা যায় না; (খ) নতুন ভাষা যোগ করলে প্রতিটা এমন ternary খুঁজে বদলাতে হয় (হিন্দি হলে `? :` আর কাজ করবে না); (গ) type-safety নেই। `t("key")` approach সব অনুবাদ কেন্দ্রীভূত রাখে, নতুন ভাষায় scalable, ও type-safe। এই project-এ inline ternary সম্ভবত দ্রুত development-এর ফল; production-grade করতে হলে সেগুলোকেও ধীরে ধীরে `t()`-এ সরিয়ে আনা উচিত। মূল নীতি — একটাই ধারাবাহিক i18n পদ্ধতি ব্যবহার করা।

### ১০. তারিখ ও সময়ের localization এখানে কীভাবে করা হয়, আর এটা কেন আলাদা মনোযোগ দাবি করে?

**উত্তর:** সংখ্যা/মুদ্রার জন্য custom helper (`formatNum`, `formatCurrency`) আছে, আর তারিখের জন্য কোথাও কোথাও `toLocaleString(lang === "bn" ? "bn-BD" : "en-GB")`-এর মতো JavaScript-এর built-in locale API ব্যবহার হয়। তারিখ/সময় আলাদা মনোযোগ দাবি করে কারণ locale-ভেদে অনেক কিছু বদলায় — সংখ্যার ভাষা (১২ vs 12), তারিখের ক্রম (DD/MM vs MM/DD), মাসের নাম, AM/PM vs 24-ঘণ্টা, এমনকি সপ্তাহ শুরুর দিন। ভুল হলে user বিভ্রান্ত হয় (যেমন 03/04 — মার্চ না এপ্রিল?)। তাই তারিখ কখনো হাতে string জোড়া না দিয়ে `Intl.DateTimeFormat`/`toLocaleString`-এর মতো locale-aware API দিয়ে format করা উচিত, যা এই সব পার্থক্য নিজে সামলায়।
