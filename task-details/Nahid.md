# Nahid – Task Details

**Focus:** User login, registration, dashboard ও রিলেটেড ইউজার ফিচার  
**Minimum:** ৩টি কমপ্লিট মেজর ফিচার (Frontend + Backend)

---

## Feature 1: User Registration & Login (Auth)

**Problem:** সিকিউর অ্যাক্সেস ও ইউজার আইডেন্টিফিকেশন দরকার।

**Solution:** রেজিস্ট্রেশন, লগইন, জWT; optional ফরগট পাসওয়ার্ড।

**Backend:** User model; `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`; জWT।  
**Frontend:** সাইনআপ/লগইন পেজ, জWT স্টোর, প্রটেক্টেড রাউট; optional ফরগট পাসওয়ার্ড।

---

## Feature 2: User Dashboard (Post-Login)

**Problem:** লগইনের পর সেন্টার পয়েন্ট নেই।

**Solution:** অ্যাক্টিভ রোডম্যাপ, কুইক অ্যাকশন, প্রোগ্রেস সামারি, রিকমেন্ডেড কোর্স।

**Backend:** `GET /api/user/dashboard` (অথবা আলাদা APIs)।  
**Frontend:** ড্যাশবোর্ড পেজ – ওয়েলকাম, স্ট্যাট, কুইক অ্যাকশন, প্রোগ্রেস, রিকমেন্ডেড কোর্স।

---

## Feature 3: User Profile, Settings & Notifications

**Problem:** প্রোফাইল এডিট, নোটিফিকেশন সেটিং ও লিস্ট নেই।

**Solution:** প্রোফাইল আপডেট, সেটিংস (নোটিফিকেশন অন/অফ), নোটিফিকেশন লিস্ট ও read স্ট্যাটাস।

**Backend:** `GET/PUT /api/user/profile`, `GET/PUT /api/user/settings`; Notification model; `GET /api/user/notifications`, mark read।  
**Frontend:** প্রোফাইল পেজ, সেটিংস পেজ, নোটিফিকেশন পেজ।

---

**চেকলিস্ট:** [ ] Feature 1  [ ] Feature 2  [ ] Feature 3  
**গ্লোবাল রিডমি:** [../README.md](../README.md)
