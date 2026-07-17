# 01 — Firebase Authentication & Auth Context

## এই ফিচারটা আসলে কী?

SkillVoyager.AI-তে ইউজার তিনভাবে login করতে পারে — **email/password**, **Google**, আর **GitHub**। এই পুরো authentication কাজটা করা হয়েছে **Firebase Authentication** দিয়ে, অর্থাৎ password hashing, session, OAuth এসব ঝামেলা নিজেদের সার্ভারে করতে হয় না — Firebase-ই সামলায়।

কিন্তু শুধু login করালেই তো হবে না — অ্যাপের যেকোনো জায়গা থেকে জানতে হবে "এখন কে logged in আছে, সে admin কিনা, data এখনো load হচ্ছে কিনা"। এই তথ্যটা প্রতিটা component-এ prop হিসেবে পাঠানো কষ্টকর। তাই এখানে **React Context API** (`AuthProvider`) ব্যবহার করা হয়েছে — একবার top-level এ auth state রাখা হয়, আর যেকোনো component `useContext(AuthContext)` দিয়ে সেটা পড়তে পারে।

এই ফিচারটা পুরো অ্যাপের ভিত্তি — কারণ Dashboard, Onboarding, Roadmap, Quiz সব protected feature এই auth state-এর উপর নির্ভর করে। তাই সবার আগে এটা বোঝা দরকার।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| Config | `frontend/src/firebase/firebase.init.js` | Firebase SDK initialize করে, `auth` object export করে |
| State/Core | `frontend/src/providers/AuthProvider.jsx` | সব auth ফাংশন + global user state ধরে রাখে (Context) |
| UI | `frontend/src/components/Login.jsx` | Login form, `signInUser`/`signInWithGoogle` কল করে |
| UI | `frontend/src/components/Register.jsx` | Sign-up form, `createUser` কল করে |
| Bridge | `backend/server.js` (`POST /api/users`) | Firebase user-কে MongoDB-তে save করে role fetch করে |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`frontend/src/firebase/firebase.init.js`** — এটা Firebase app-কে environment variable (`VITE_FIREBASE_*`) দিয়ে initialize করে এবং `getAuth(app)` থেকে পাওয়া `auth` instance export করে; পুরো অ্যাপে ঠিক একটাই Firebase auth instance থাকে বলে এই ফাইল আছে।
- **`frontend/src/providers/AuthProvider.jsx`** — এটাই ফিচারের হৃদয়: register/login/logout/Google/GitHub সব ফাংশন এখানে, আর `onAuthStateChanged` দিয়ে user state সবসময় sync রাখে।
- **`frontend/src/components/Login.jsx` / `Register.jsx`** — শুধু UI form; আসল কাজ context-এর ফাংশনগুলোকে delegate করে, তাই auth logic এক জায়গায় থাকে।
- **`backend/server.js` → `POST /api/users`** — Firebase শুধু "কে" জানে; কিন্তু আমাদের নিজেদের DB-তে points, role, onboarding ইত্যাদি রাখতে হয়, তাই login-এর সময় এই endpoint Firebase user-কে MongoDB-তে upsert করে।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — Firebase একবার initialize করা

`firebase.init.js` env variable থেকে config নিয়ে একটা মাত্র `auth` instance বানায়। `import.meta.env.VITE_*` হলো Vite-এর way — build-time এ এই values inject হয়।

```js
// frontend/src/firebase/firebase.init.js
const firebaseConfig = {
    apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
    authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
    projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
    // ...
};
const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);   // ⬅️ পুরো অ্যাপ এই একটাই instance ব্যবহার করবে
```

### ধাপ ২ — AuthProvider সব auth ফাংশন সরবরাহ করে

`createUser`, `signInUser`, `signInWithGoogle`, `logOut` — সব Firebase call এখানে wrap করা। খেয়াল করুন প্রতিটা call-এর আগে `setLoading(true)` করা হচ্ছে, যাতে UI "loading" দেখাতে পারে।

```jsx
// frontend/src/providers/AuthProvider.jsx
const createUser = (email, password) => {
    setLoading(true);
    return createUserWithEmailAndPassword(auth, email, password);
}

const signInWithGoogle = () => {
    setLoading(true);
    return signInWithPopup(auth, googleProvider);   // popup — localhost এ redirect-এর চেয়ে reliable
}
```

### ধাপ ৩ — `onAuthStateChanged` দিয়ে state সবসময় sync

এটাই সবচেয়ে গুরুত্বপূর্ণ অংশ। Firebase একটা **observer** দেয় — user login/logout করলেই এই callback চলে। এখানে user state set হয়, backend-এ user save হয়, আর backend থেকে `role` সহ `dbUser` আনা হয়।

```jsx
// frontend/src/providers/AuthProvider.jsx
useEffect(() => {
    const unSubscribe = onAuthStateChanged(auth, async (currentUser) => {
        setUser(currentUser);
        if (currentUser) {
            const res = await fetch(`${API_BASE}/api/users`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ uid: currentUser.uid, email: currentUser.email, /* ... */ }),
            });
            const data = await res.json();
            if (data.success) setDbUser(data.user);   // ⬅️ role এখান থেকেই আসে
        } else {
            setDbUser(null);
        }
        setLoading(false);
    });
    return () => unSubscribe();   // ⬅️ cleanup: memory leak ঠেকায়
}, [])
```

গুরুত্বপূর্ণ লাইন: `return () => unSubscribe()` — component unmount হলে observer বন্ধ করে দেয়।

### ধাপ ৪ — Context দিয়ে সবাইকে share করা

শেষে সব ফাংশন + state একটা object-এ ভরে `AuthContext.Provider`-এ দেওয়া হয়। এখন যেকোনো component `useContext(AuthContext)` দিয়ে `user`, `dbUser`, `loading` পাবে।

```jsx
const authInfo = { user, dbUser, loading, createUser, signInUser, signInWithGoogle, signInWithGithub, logOut, sendPasswordReset };
return <AuthContext.Provider value={authInfo}>{children}</AuthContext.Provider>;
```

### ধাপ ৫ — একটা লুকোনো ডিটেইল: auto-provision admin

`signInUser`-এ একটা বিশেষ কেস আছে — নির্দিষ্ট admin email প্রথমবার login করলে account না থাকলে নিজে থেকে বানিয়ে নেয়:

```jsx
// frontend/src/providers/AuthProvider.jsx
if (email === "admin@skillvoyager.ai" && password === "CodeCatalysts" &&
    (error.code === 'auth/user-not-found' || error.code === 'auth/invalid-credential' /* ... */)) {
    return await createUserWithEmailAndPassword(auth, email, password);   // auto-provision
}
```

> ⚠️ **সততার সাথে:** admin email আর password এভাবে frontend code-এ hardcode করা একটা demo-shortcut এবং নিরাপত্তার দিক থেকে দুর্বল — এটা bundle-এ চলে যায়, যে কেউ দেখতে পারে। Production-এ admin role backend থেকে নিয়ন্ত্রণ করা উচিত, code-এ credential রাখা উচিত না।

## মূল ধারণা (Key Concepts)

- **BaaS (Backend-as-a-Service)** — Firebase নিজেই auth, password, OAuth সামলায়; আমরা শুধু SDK কল করি।
- **Observer pattern** — `onAuthStateChanged` একটা subscription; state বদলালে callback চলে, আমরা poll করি না।
- **React Context API** — prop-drilling ছাড়া global state (এখানে auth) সব component-এ পৌঁছানো।
- **`loading` state** — auth resolve হওয়ার আগ পর্যন্ত UI-কে অপেক্ষা করানো, যাতে ভুল করে "logged out" না দেখায়।
- **Cleanup function** — `useEffect`-এর return-এ unsubscribe করা, নাহলে listener জমে memory leak হয়।
- **Provider bridging** — Firebase (identity) + নিজের DB (business data) দুটোকে একসাথে জোড়া লাগানো।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. Firebase Auth ব্যবহার করলেন কেন, নিজের JWT-based auth না বানিয়ে?
**উত্তর:** Authentication ঠিকভাবে বানানো কঠিন — password hashing, salting, secure session, password reset email, rate limiting, OAuth flow (Google/GitHub), token refresh — এসব নিজে বানালে ভুল হওয়ার সম্ভাবনা বেশি এবং সময় লাগে। Firebase এসব battle-tested ভাবে দেয়, আর Google/GitHub sign-in মাত্র কয়েক লাইনে হয়ে যায় (`signInWithPopup`)। SkillVoyager একটা learning-focused product, তাই আমরা auth নিয়ে সময় নষ্ট না করে core feature (roadmap, quiz, progress)-এ ফোকাস করেছি। Trade-off হলো — vendor lock-in আর backend-এ token verify করার দায়িত্ব; আমাদের ক্ষেত্রে সেই দ্বিতীয় অংশটা এখনো fully implemented না, যেটা একটা দুর্বলতা।

### ২. `onAuthStateChanged`-এর ভেতরে backend call করা হচ্ছে — এটা `useEffect`-এ না রেখে login ফাংশনে রাখলে সমস্যা কী হতো?
**উত্তর:** `onAuthStateChanged` শুধু login-এ চলে না — page refresh করলেও Firebase auth state restore হয় এবং তখন এই observer আবার চলে। যদি backend sync শুধু `signInUser` ফাংশনে রাখতাম, তাহলে refresh-এর পর user logged in থাকলেও `dbUser` (role) আসত না, ফলে refresh-এর পর admin আর admin হিসেবে চিনত না। Observer-এ রাখায় "যেকোনোভাবেই user resolve হোক" — login হোক বা refresh — একই জায়গা থেকে DB sync হয়। এটাই single source of truth করার সুবিধা।

### ৩. `unSubscribe` cleanup না করলে বাস্তবে কী ভাঙবে?
**উত্তর:** `AuthProvider` mount হলে একটা listener register হয়। Cleanup না থাকলে, প্রতিবার component re-mount (বা dev-mode-এ React StrictMode-এর double mount)-এ নতুন listener জমতে থাকবে, পুরনোগুলো মরবে না। এতে একই auth event-এ একাধিকবার `setUser`/backend call হবে, memory leak হবে, আর duplicate network request যাবে। যেহেতু `AuthProvider` root-এ একবারই থাকে, বাস্তবে impact ছোট, কিন্তু pattern হিসেবে cleanup না করা bug — বিশেষ করে StrictMode-এ dev-এ double effect চলে বলে এটা ধরা পড়ে।

### ৪. Google/GitHub-এ একই email দিয়ে account থাকলে কী হয়, আপনি কীভাবে handle করলেন?
**উত্তর:** Firebase default-এ এক email = এক account নীতি মানে। কেউ আগে Google দিয়ে sign in করে থাকলে পরে সেই একই email-এ GitHub দিয়ে চেষ্টা করলে `auth/account-exists-with-different-credential` error আসে। আমি `signInWithGithub`-এ এই error ধরে `fetchSignInMethodsForEmail` দিয়ে দেখি email-টা কোন method-এ আছে (google.com নাকি password), তারপর user-কে পরিষ্কার message দিই — "আগে Google দিয়ে login করে তারপর GitHub connect করুন"। এতে user একটা cryptic Firebase error না দেখে বুঝতে পারে কী করতে হবে। Alternative হতো account linking automatically করা, কিন্তু সেটা আরও state ও credential handling চায়।

### ৫. `user` আর `dbUser` — দুটো আলাদা রাখলেন কেন?
**উত্তর:** `user` হলো Firebase-এর identity object (uid, email, photoURL) — এটা "কে" বলে। `dbUser` হলো আমাদের MongoDB-র record — এটা business data বলে (role, points, streak, onboarding)। এদের আলাদা রাখার কারণ, দুটোর উৎস আলাদা এবং update pattern আলাদা: Firebase user Firebase বদলালে বদলায়, `dbUser` আমাদের API বদলালে। যেমন `PrivateRoute`-এ admin চেক করতে `dbUser.role` লাগে যেটা Firebase-এ নেই। একসাথে merge করে ফেললে কোনটা authoritative সেটা গুলিয়ে যেত।

### ৬. `loading` state না থাকলে কী সমস্যা হতো?
**উত্তর:** App load হওয়ার প্রথম মুহূর্তে Firebase এখনো জানে না user logged in কিনা — `onAuthStateChanged` async ভাবে resolve হয়। `loading` না থাকলে সেই ফাঁকা মুহূর্তে `user` হবে `null`, ফলে `PrivateRoute` মনে করবে user logged out এবং সাথে সাথে `/login`-এ পাঠিয়ে দেবে — অথচ user আসলে logged in। এতে refresh করলেই protected page থেকে ছিটকে যেত। `loading` true থাকা অবস্থায় আমরা "Loading..." দেখাই এবং auth resolve না হওয়া পর্যন্ত redirect decision নিই না।

### ৭. Frontend-এ hardcoded admin credential রাখাটা কতটা বিপজ্জনক, production-এ কী করতেন?
**উত্তর:** খুব বিপজ্জনক — Vite build করলে এই string bundle-এ যায়, browser dev-tools খুলে যে কেউ admin password দেখতে পারবে। এটা একটা demo shortcut, দ্রুত একটা admin account বানানোর জন্য। Production-এ আমি (ক) admin account আগে থেকে Firebase-এ manually বা secure script দিয়ে বানাতাম, (খ) role assignment হতো backend-এ — যেমন Firebase **custom claims** বা DB-তে trusted list, (গ) কোনো credential কখনো client code-এ থাকত না। এমনকি এখনকার `/api/users`-এও `email === 'admin@skillvoyager.ai'` হলে role=admin দেওয়া হয়, যেটা spoof-yোগ্য কারণ email verify হয় না — এটাও শক্ত করতে হতো।

### ৮. এই auth system-এ backend endpoint গুলো কীভাবে সুরক্ষিত? কোনো ফাঁক আছে?
**উত্তর:** সততার সাথে — বড় একটা ফাঁক আছে। Frontend Firebase দিয়ে login করে ঠিকই, কিন্তু backend-এ request আসার সময় আমরা Firebase **ID token verify করি না**। বেশিরভাগ endpoint `uid` বা `email`-কে body/param থেকে বিশ্বাস করে নেয়। মানে কেউ চাইলে অন্যের `uid` পাঠিয়ে তার data পড়তে/বদলাতে পারে। সঠিক approach হতো: frontend প্রতিটা request-এ `Authorization: Bearer <idToken>` পাঠাবে, backend `firebase-admin` SDK দিয়ে `verifyIdToken` করে uid বের করবে — client-এর পাঠানো uid বিশ্বাস করবে না। এটা এই প্রজেক্টের সবচেয়ে গুরুত্বপূর্ণ hardening।

### ৯. `signInWithPopup` বনাম `signInWithRedirect` — কোনটা কেন?
**উত্তর:** কোডে comment-এই লেখা আছে "popup — more reliable on localhost than redirect"। Redirect পুরো পেজ reload করে OAuth provider-এ পাঠায় এবং ফিরে আসার পর `getRedirectResult` handle করতে হয়, যেটা localhost/third-party cookie সেটিংয়ে মাঝে মাঝে ঝামেলা করে। Popup একটা আলাদা window খোলে, main app state নষ্ট হয় না, আর flow সহজ (promise resolve করলেই user পাওয়া যায়)। Trade-off: mobile-এ বা popup-blocker থাকলে popup আটকে যেতে পারে, তখন redirect বেশি নির্ভরযোগ্য — বড় স্কেলে দুটোর fallback রাখা ভালো।

### ১০. এই Context approach কি Redux/Zustand-এর চেয়ে ভালো? কখন migrate করতেন?
**উত্তর:** এই ক্ষেত্রে Context-ই যথেষ্ট এবং ভালো, কারণ auth state ছোট (user, dbUser, loading + কয়েকটা function) এবং কদাচিৎ বদলায়। এর জন্য Redux আনলে boilerplate বাড়ত, benefit কম। Context-এর দুর্বলতা হলো — value বদলালে সব consumer re-render হয়; যদি এই provider-এ ঘন ঘন বদলানো বড় state (যেমন real-time chat, live progress) ঢুকত, তখন unnecessary re-render performance সমস্যা করত। সেই পর্যায়ে Zustand/Redux বা context-কে ভাগ করা (auth context আলাদা, data context আলাদা) দরকার হতো। এখনকার scale-এ migrate করার দরকার নেই — premature optimization হতো।
