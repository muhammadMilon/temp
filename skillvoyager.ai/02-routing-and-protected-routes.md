# 02 — Routing, Layout & Protected Routes (RBAC)

## এই ফিচারটা আসলে কী?

SkillVoyager একটা **SPA (Single Page Application)** — অর্থাৎ পুরো অ্যাপ একবারই load হয়, তারপর page বদলালেও সার্ভার থেকে নতুন HTML আসে না, শুধু URL আর component বদলায়। এই client-side routing করা হয়েছে **React Router v7**-এর `createBrowserRouter` দিয়ে।

কিন্তু সব page সবার জন্য না। `/dashboard`, `/progress`, `/onboarding` — এগুলো শুধু logged-in user দেখতে পারবে; `/admin-dashboard` শুধু **admin**। এই "কে কোন page দেখতে পারবে" নিয়ন্ত্রণটাই **RBAC (Role-Based Access Control)**, আর এটা করা হয়েছে একটা ছোট wrapper component `PrivateRoute` দিয়ে।

এছাড়া প্রায় সব page-এর উপরে Navbar আর নিচে Footer একই থাকে। এই common shell বারবার না লিখে একটা **layout route** (`MainLayout`) বানানো হয়েছে, যার ভেতরে `<Outlet />` দিয়ে আসল page render হয়।

## কোন কোন ফাইল জড়িত?

| স্তর (Layer) | ফাইল (File) | ভূমিকা (Role) |
|--------------|-------------|----------------|
| Router | `frontend/src/main.jsx` | সব route define করে, provider দিয়ে অ্যাপ wrap করে |
| Layout | `frontend/src/layouts/MainLayout.jsx` | Navbar + `<Outlet/>` + Footer এর common shell |
| Guard | `frontend/src/components/PrivateRoute.jsx` | Login/role চেক করে access দেয় বা redirect করে |
| State | `frontend/src/providers/AuthProvider.jsx` | `user`, `dbUser.role`, `loading` সরবরাহ করে (গার্ডের ইনপুট) |

### জড়িত ফাইলগুলো, সংক্ষেপে

- **`frontend/src/main.jsx`** — এখানে `createBrowserRouter` দিয়ে route tree বানানো, আর `RouterProvider`, `AuthProvider`, `ThemeProvider`, `ToastContainer` দিয়ে root render হয়; পুরো navigation-এর নকশা এই এক ফাইলে।
- **`frontend/src/layouts/MainLayout.jsx`** — parent route হিসেবে বসে, Navbar/Footer একবার render করে আর মাঝখানে `<Outlet/>`-এ child page দেখায়; DRY রাখার জন্য।
- **`frontend/src/components/PrivateRoute.jsx`** — একটা guard: `loading`, `user`, `adminOnly` দেখে হয় children দেখায়, নাহলে `/login` বা `/dashboard`-এ redirect করে।

## পুরো ফ্লো টা কীভাবে কাজ করে

### ধাপ ১ — Route tree বানানো (`createBrowserRouter`)

`main.jsx`-এ একটা array দিয়ে পুরো routing define করা। লক্ষ্য করুন: root path `/`-এর element হলো `MainLayout`, আর বাকি সব page তার `children` — এটাই **nested routing**।

```jsx
// frontend/src/main.jsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <MainLayout />,        // ⬅️ shell (Navbar + Footer)
    children: [
      { path: "/", element: <App /> },              // landing page
      { path: "/login", element: <Login /> },       // public
      { path: "/register", element: <Register /> }, // public
      { path: "/dashboard", element: <PrivateRoute><Dashboard /></PrivateRoute> },
      { path: "/admin-dashboard", element: <PrivateRoute adminOnly><AdminDashboard /></PrivateRoute> },
      // ...
      { path: "*", element: /* 404 page */ }         // ⬅️ catch-all
    ]
  }
]);
```

গুরুত্বপূর্ণ: `path: "*"` হলো catch-all — কোনো route না মিললে custom 404 ("Voyage Interrupted!") দেখায়।

### ধাপ ২ — Layout route + `<Outlet/>`

`MainLayout` Navbar আর Footer render করে, মাঝখানে `<Outlet/>` বসায়। React Router যে child route match করে, সেটা এই `Outlet`-এর জায়গায় বসিয়ে দেয়।

```jsx
// frontend/src/layouts/MainLayout.jsx
const NO_FOOTER_ROUTES = ['/admin-dashboard'];
const MainLayout = () => {
    const { pathname } = useLocation();
    const showFooter = !NO_FOOTER_ROUTES.includes(pathname);   // admin page এ footer লুকানো
    return (
        <div className="flex flex-col min-h-screen">
            <Navbar />
            <main className="flex-grow"><Outlet /></main>   {/* ⬅️ child page এখানে বসে */}
            {showFooter && <Footer />}
        </div>
    );
};
```

### ধাপ ৩ — `PrivateRoute` দিয়ে access control

এই ছোট component-টাই RBAC-এর পুরো logic ধরে। তিনটা সিদ্ধান্ত ক্রমে নেয়: (১) এখনো loading? তাহলে অপেক্ষা করো। (২) user নেই? `/login`। (৩) admin-only route কিন্তু user admin না? `/dashboard`।

```jsx
// frontend/src/components/PrivateRoute.jsx
const PrivateRoute = ({ children, adminOnly = false }) => {
    const { user, dbUser, loading } = useContext(AuthContext);

    if (loading) return <div>...Loading...</div>;          // ⬅️ (১) decision দেরি করাও
    if (!user) return <Navigate to="/login" />;             // ⬅️ (২) logged out
    if (adminOnly && dbUser?.role !== 'admin')              // ⬅️ (৩) role চেক
        return <Navigate to="/dashboard" />;

    return children;   // সব পাস → আসল page দেখাও
};
```

গুরুত্বপূর্ণ লাইন: `if (loading) return ...` — এটা প্রথমে থাকা জরুরি, নাহলে auth resolve হওয়ার আগেই logged-in user-কে ভুল করে বের করে দিত (দেখুন [01-firebase-auth](01-firebase-auth.md) প্রশ্ন ৬)।

### ধাপ ৪ — ব্যবহার

Route define করার সময় শুধু page-টাকে `PrivateRoute` দিয়ে wrap করলেই হয়:

```jsx
{ path: "/progress",       element: <PrivateRoute><ProgressDashboard /></PrivateRoute> }
{ path: "/admin-dashboard",element: <PrivateRoute adminOnly><AdminDashboard /></PrivateRoute> }
```

`adminOnly` prop না দিলে default `false` — শুধু login লাগবে; দিলে role=admin-ও লাগবে।

## মূল ধারণা (Key Concepts)

- **Client-side routing (SPA)** — URL বদলালে সার্ভার hit না করে JS component swap করা।
- **Nested routes + `<Outlet/>`** — parent layout একবার, child page গুলো তার ভেতরে render হয় → DRY।
- **Higher-order / wrapper component** — `PrivateRoute` children-কে ঘিরে ধরে condition অনুযায়ী গার্ড করে।
- **Declarative redirect** — `<Navigate to=... />` render করলেই React Router redirect করে।
- **RBAC** — একই guard-এ `adminOnly` flag দিয়ে ভিন্ন role-এর access আলাদা করা।
- **Catch-all route (`*`)** — unmatched URL-এর জন্য 404 fallback।

## ইন্টারভিউ প্রশ্ন (Interview Questions)

### ১. `PrivateRoute`-এ চেকগুলোর order (loading → user → role) বদলে দিলে কী ভাঙবে?
**উত্তর:** Order-টাই এখানে সবচেয়ে সূক্ষ্ম অংশ। `loading` চেক যদি সবার আগে না থাকে, তাহলে প্রথম render-এ `user` এখনো `null` (Firebase resolve করেনি), ফলে `!user` true হয়ে logged-in user-ও `/login`-এ চলে যাবে — refresh করলেই protected page থেকে ছিটকে যাওয়ার bug। আবার `!user` চেক যদি `adminOnly` চেকের পরে রাখতাম, তাহলে `dbUser?.role` পড়ার সময় user null হলেও crash হতো না (optional chaining আছে), কিন্তু logic এলোমেলো হতো। তাই order হলো: আগে অপেক্ষা করো (loading), তারপর "সে কি আদৌ কেউ?" (user), তারপর "সে কি admin?" (role)।

### ২. এই RBAC পুরোপুরি frontend-এ — এটা কি যথেষ্ট? Admin page কি সত্যিই সুরক্ষিত?
**উত্তর:** না, যথেষ্ট না — এবং এটা এই আর্কিটেকচারের একটা বড় দুর্বলতা। `PrivateRoute` শুধু **UI hide** করে; এটা security boundary না। কেউ `dbUser.role` কে dev-tools দিয়ে বদলে দিলে, বা সরাসরি admin API endpoint-এ request পাঠালে, frontend guard কিছুই আটকাবে না — কারণ backend endpoint গুলো role verify করে না। সঠিক নিরাপত্তা হলো: প্রতিটা sensitive backend route-এ middleware দিয়ে Firebase ID token verify করা এবং সেখানে role চেক করা। Frontend guard শুধু UX — সাধারণ user যেন ভুল page না দেখে; আসল enforcement সবসময় server-side হতে হবে।

### ৩. Layout route ব্যবহার করলেন কেন, প্রতিটা page-এ আলাদা করে Navbar/Footer না দিয়ে?
**উত্তর:** DRY (Don't Repeat Yourself)। ৩০+ page-এর প্রতিটায় `<Navbar/>` আর `<Footer/>` import করে বসালে code duplicate হতো, আর Navbar-এ পরিবর্তন করতে গেলে সব page ঘুরে ঠিক করতে হতো। `MainLayout`-কে parent route বানিয়ে `<Outlet/>` দেওয়ায় shell একবার লেখা হয়, সব child তার ভেতরে বসে। বোনাস — page transition-এ Navbar re-mount হয় না, তাই state (যেমন notification dropdown) নষ্ট হয় না এবং performance ভালো।

### ৪. `MainLayout`-এ `NO_FOOTER_ROUTES` — এটা scale করবে? আরও ভালো উপায় কী?
**উত্তর:** এখন এটা একটা hardcoded array (`['/admin-dashboard']`) আর `pathname` মিলিয়ে দেখে। ছোট অ্যাপে ঠিক আছে, কিন্তু route বাড়লে এই list maintain করা বিরক্তিকর এবং ভুল হওয়ার সুযোগ। ভালো approach হতো — nested layout ব্যবহার করা: admin section-এর জন্য আলাদা `AdminLayout` (Navbar আছে, Footer নেই) বানিয়ে admin route গুলো তার children করা। তাহলে "footer আছে/নেই" route config থেকেই ঠিক হতো, string-match লাগত না। React Router v7-এ এই nested layout প্যাটার্ন সহজেই করা যায়।

### ৫. React Router v7-এর `createBrowserRouter` আর পুরনো `<BrowserRouter><Routes>` এর মধ্যে পার্থক্য কী?
**উত্তর:** `createBrowserRouter` হলো data-router API — route গুলো একটা object/array হিসেবে define হয় (JSX tree হিসেবে না), যা loader, action, error boundary-এর মতো data-focused feature unlock করে। এতে routing "data-first" হয়, শুধু rendering না। পুরনো `<Routes>` approach declarative JSX ভিত্তিক, সহজ কিন্তু loader/action pattern নেই। এই প্রজেক্টে loader/action এখনো ব্যবহার করা হয়নি (data fetch component-এর `useEffect`-এ হচ্ছে), তাই `createBrowserRouter`-এর পূর্ণ সুবিধা এখনো নেওয়া হয়নি — ভবিষ্যতে data loader দিয়ে fetch করলে code আরও পরিষ্কার হবে।

### ৬. Catch-all `path: "*"` route না থাকলে ভুল URL-এ কী হতো?
**উত্তর:** কোনো route match না করলে React Router কিছুই render করত না — user একটা ফাঁকা/ভাঙা page দেখত, কোনো feedback ছাড়া। `path: "*"` সবচেয়ে শেষে থাকায় এটা "যা কিছু বাকি" ধরে এবং একটা বন্ধুত্বপূর্ণ 404 ("Voyage Interrupted!") + Dashboard-এ ফেরার link দেখায়। এটা UX-এর জন্য জরুরি — typo করা URL, expire হওয়া link, বা ভুল bookmark সব এখানে gracefully handle হয়।

### ৭. `<Navigate to="/login" />` দিয়ে redirect করার সময় user কোথায় যেতে চেয়েছিল সেটা মনে রাখা হয় কি? না হলে সমস্যা কী?
**উত্তর:** এখন মনে রাখা হয় না — logged-out user `/progress`-এ যেতে চাইলে `/login`-এ পাঠানো হয়, কিন্তু login-এর পর সে `/progress`-এ ফেরে না, dashboard বা home-এ যায়। এটা একটা UX ঘাটতি। ভালো practice হলো redirect করার সময় intended path সংরক্ষণ করা — যেমন `<Navigate to="/login" state={{ from: location }} />`, তারপর login সফল হলে `state.from`-এ ফিরিয়ে দেওয়া। এতে deep link শেয়ার করলে বা session expire হলে user login-এর পর ঠিক যেখানে যেতে চেয়েছিল সেখানেই যায়।

### ৮. একই `PrivateRoute` দিয়ে দুই ধরনের role handle করছেন `adminOnly` prop দিয়ে — role বাড়লে (যেমন mentor, moderator) এটা কেমন হবে?
**উত্তর:** এখনকার boolean `adminOnly` দুই role (user/admin)-এর জন্য যথেষ্ট এবং সহজ। কিন্তু role বাড়লে boolean আর কাজ করবে না — mentor-only, moderator-or-admin এসব দরকার হবে। তখন আমি prop-টা পাল্টে `allowedRoles={['admin','mentor']}` array করতাম এবং চেক করতাম `allowedRoles.includes(dbUser?.role)`। এতে একই guard যেকোনো role-combination handle করত, নতুন role-এ শুধু array বদলালেই হতো, নতুন component লাগত না। এটা open/closed principle-এর কাছাকাছি — বাড়ানো যায় কিন্তু guard-এর core বদলায় না।

### ৯. Route-level code splitting (lazy loading) করলে এখানে কী সুবিধা হতো?
**উত্তর:** এখন সব page (`main.jsx`-এর সব import) app শুরুতেই একসাথে bundle হয়ে load হয়, ফলে initial bundle বড় ও প্রথম load ধীর — অথচ user হয়তো Admin dashboard বা PDF generator কখনোই খুলবে না। `React.lazy()` + `Suspense` দিয়ে প্রতিটা route-এর component আলাদা chunk করে দিলে, শুধু দরকারের সময় সেই chunk download হতো। বিশেষ করে ভারী dependency (chart.js, @react-pdf/renderer, framer-motion, three) যেসব page-এ আছে সেগুলো lazy করলে landing page-এর load time নাটকীয়ভাবে কমত। এটা এই প্রজেক্টে একটা সহজ ও কার্যকর optimization।

### ১০. `main.jsx`-এ Provider-দের nesting order (ThemeProvider → AuthProvider → RouterProvider) কি গুরুত্বপূর্ণ?
**উত্তর:** হ্যাঁ, order মানে dependency। ভেতরের component-রা বাইরের provider-এর context পড়তে পারে, উল্টোটা না। `AuthProvider`-কে `RouterProvider`-এর বাইরে রাখা হয়েছে যাতে সব route (এবং `PrivateRoute`) auth context পায় — এটা অপরিহার্য, কারণ guard গুলো auth-এর উপর নির্ভর করে। `ThemeProvider` সবার বাইরে, কারণ theme পুরো অ্যাপ জুড়ে লাগে এবং কারো উপর নির্ভর করে না। যদি `RouterProvider`-কে `AuthProvider`-এর বাইরে রাখতাম, তাহলে route-এর ভেতরের component `useContext(AuthContext)` করলে `null` পেত এবং auth-নির্ভর সব guard ভেঙে পড়ত।
