# Kakoly – Task Details

**Focus:** UI-heavy full features (প্রতিটি ফিচার Frontend + Backend কমপ্লিট)  
**Minimum:** ৩টি কমপ্লিট মেজর ফিচার

---

## Feature 1: Progress Tracking Dashboard (Full Stack)

**Problem:** প্রোগ্রেস, মাইলস্টোন বাকি, এস্টিমেটেড টাইম এক জায়গায় নেই।

**Solution:** রিয়েল ডেটা দিয়ে ড্যাশবোর্ডে প্রোগ্রেস চার্ট/বার, মাইলস্টোন স্ট্যাটাস, এস্টিমেটেড দিন।

**Backend:** `GET /api/user/progress`, `GET /api/user/progress/summary`।  
**Frontend:** ড্যাশবোর্ড – স্ট্যাট কার্ড, প্রোগ্রেস বার, (optional) চার্ট, API ডেটা।

---

## Feature 2: Learning Tips & Resources Section (Full Stack)

**Problem:** টিপস/রিসোর্স স্ট্যাটিক; আপডেট ও ইউজার বুকমার্ক দরকার।

**Solution:** টিপস/রিসোর্স DB-তে; API; (optional) বুকমার্ক।

**Backend:** `GET /api/tips`, `GET /api/resources`; optional `POST/GET /api/user/bookmarks`।  
**Frontend:** টিপস পেজ, রিসোর্স পেজ/ট্যাব, optional Save for later।

---

## Feature 3: Onboarding Flow & Profile Setup (Full Stack)

**Problem:** নিউ ইউজার গোল/স্কিল/টাইমলাইন একসেটাপে দেয় না।

**Solution:** মাল্টি-স্টেপ অনবোর্ডিং (role, education, skills, target career, timeline); ডেটা সেভ করে প্রোফাইল/রোডম্যাপে ব্যবহার।

**Backend:** `POST /api/user/onboarding` – steps সেভ।  
**Frontend:** স্টেপ ইন্ডিকেটর, ৩ স্টেপ ফর্ম, Complete → ড্যাশবোর্ড/রোডম্যাপ।

---

**চেকলিস্ট:** [ ] Feature 1  [ ] Feature 2  [ ] Feature 3  
**গ্লোবাল রিডমি:** [../README.md](../README.md)
