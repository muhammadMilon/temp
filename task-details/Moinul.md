# Moinul – Task Details

**Focus:** Seed admin, Admin dashboard, Payment system integration  
**Minimum:** ৩টি কমপ্লিট মেজর ফিচার (Frontend + Backend, Problem → Solution)

---

## Feature 1: Seed Data & Admin User

**Problem:** ডেভ/টেস্ট করার সময় ইউজার, রোডম্যাপ, কোর্স, টিপস ইত্যাদি ডেটা এবং অ্যাডমিন অ্যাকাউন্ট দরকার; ম্যানুয়াল এন্ট্রি সময় নেয়।

**Solution:** সিড স্ক্রিপ্ট যা রান করলে অ্যাডমিন ইউজার, স্যাম্পেল ইউজার, স্যাম্পেল রোডম্যাপ/মাইলস্টোন, টিপস/রিসোর্স, ট্রেন্ডিং স্কিল ডেটা ইত্যাদি DB-তে ইনসার্ট করে।

**Backend:**
- `scripts/seed.js` (বা `npm run seed`) – MongoDB কানেক্ট করে:
  - অ্যাডমিন ইউজার (role: 'admin') কমপ্লিট প্রোফাইল ও পাসওয়ার্ড হ্যাশ
  - ২–৩টা স্যাম্পেল নরমাল ইউজার
  - স্যাম্পেল রোডম্যাপ ও মাইলস্টোন (কোন ইউজারের সাথে অ্যাসাইন)
  - টিপস ও রিসোর্স এন্ট্রি (কাকলির ফিচার এর জন্য)
  - ট্রেন্ডিং স্কিল বা স্যাম্পেল স্কিল ডেটা (optional)
- রান করার আগে চেক করা যে ডেটা already আছে কিনা (idempotent optional), অথবা ক্লিয়ার করে সিড

**Deliverables:** Working seed script, Admin user তৈরি, স্যাম্পেল ডেটা যাতে বাকি ফিচারগুলো টেস্ট করা যায়।

---

## Feature 2: Admin Dashboard

**Problem:** অ্যাডমিন ইউজার, রোডম্যাপ, কন্টেন্ট, রিপোর্ট ম্যানেজ বা মনিটর করার জায়গা নেই।

**Solution:** অ্যাডমিন-অনলি ড্যাশবোর্ড যেখানে ইউজার লিস্ট, বেসিক স্ট্যাটস (টোটাল ইউজার, রোডম্যাপ, কুইজ সাবমিশন), কন্টেন্ট মডারেশন বা টিপস/রিসোর্স ক্রুড (optional), এবং সিম্পল রিপোর্ট দেখানো।

**Backend:**
- মিডলওয়্যার: `isAdmin` – জWT থেকে role চেক, admin না হলে 403
- API: `GET /api/admin/stats` – totalUsers, totalRoadmaps, totalQuizzes, recentSignups
- API: `GET /api/admin/users` (পেজিনেশন, সার্চ optional)
- API: `GET/POST/PUT/DELETE /api/admin/tips` ও `/api/admin/resources` (কাকলির টিপস/রিসোর্স ক্রুড – অ্যাডমিন ভার্সন)
- (Optional) `GET /api/admin/reports` – সিম্পল অ্যাগ্রিগেটেড রিপোর্ট

**Frontend:**
- অ্যাডমিন লগইন বা নরমাল লগইন থেকে role চেক করে অ্যাডমিন ড্যাশবোর্ডে রিডাইরেক্ট
- অ্যাডমিন ড্যাশবোর্ড পেজ: সাইডবার/নেভ (Users, Stats, Tips, Resources, Reports)
- স্ট্যাটস কার্ড, ইউজার টেবিল (পেজিনেশন), টিপস/রিসোর্স লিস্ট ও এডিট/অ্যাড/ডিলিট ফর্ম বা মোডাল

**Deliverables:** Admin middleware, Admin APIs (stats, users, tips/resources CRUD), Admin dashboard UI।

---

## Feature 3: Payment System Integration

**Problem:** ফ্রি/প্রো প্ল্যান থাকলে ইউজার সাবস্ক্রিপশন বা ওয়ান-টাইম পেমেন্ট দিয়ে প্রো ফিচার এক্সেস করতে পারবে না।

**Solution:** স্ট্রাইপ (বা অন্য পেমেন্ট গেটওয়ে) ইন্টিগ্রেশন: প্রো প্ল্যান সিলেক্ট করলে চেকআউট সেশন তৈরি, পেমেন্ট সাকসেসে ওয়েবহুক/API দিয়ে সাবস্ক্রিপশন স্ট্যাটাস আপডেট, ফ্রন্টএন্ডে প্ল্যান ও এক্সপায়ার ডেটা দেখানো।

**Backend:**
- Subscription/Plan model: userId, planId (free/pro), status, stripeCustomerId, stripeSubscriptionId, currentPeriodEnd
- স্ট্রাইপ API কী (env), ওয়েবহুক সিক্রেট
- `POST /api/payment/create-checkout-session` – planId, success/cancel URL; স্ট্রাইপ চেকআউট সেশন ক্রিয়েট, সেশন URL রিটার্ন
- ওয়েবহুক এন্ডপয়েন্ট: `POST /api/payment/webhook` – subscription.created/updated/deleted, payment_intent.succeeded ইত্যাদি হ্যান্ডল করে Subscription DB আপডেট
- `GET /api/user/subscription` – কারেন্ট প্ল্যান, status, currentPeriodEnd

**Frontend:**
- প্রাইসিং পেজ: ফ্রি ও প্রো প্ল্যান কার্ড, “Get Started” / “Subscribe” বাটন
- প্রো সিলেক্ট করলে চেকআউট API কল → স্ট্রাইপ চেকআউট পেজে রিডাইরেক্ট
- সাকসেস/ক্যান্সেল পেজ: “Payment successful” বা “Cancelled”, ড্যাশবোর্ডে রিডাইরেক্ট
- প্রোফাইল বা সেটিংসে “Current plan” ও এক্সপায়ার ডেটা (API থেকে)

**Deliverables:** Stripe (বা সমমান) integration, checkout session, webhook, subscription status API, Pricing & success/cancel UI।

---

## টাস্ক চেকলিস্ট

- [ ] Feature 1: Seed script – Admin user + sample users, roadmaps, tips/resources
- [ ] Feature 2: Admin Dashboard – Admin APIs + Admin UI (stats, users, tips/resources)
- [ ] Feature 3: Payment – Checkout, webhook, subscription API + Pricing & success UI

**গ্লোবাল রিডমি:** [../README.md](../README.md)
