# Nayem – Task Details

**Focus:** AI-related features  
**Minimum:** ৩টি কমপ্লিট মেজর ফিচার (Frontend + Backend)

---

## Feature 1: Personalized Roadmap Generation (AI)

**Problem:** সবার একই রোডম্যাপ; গোল/টাইমলাইন/লার্নিং স্টাইল অনুযায়ী নেই।

**Solution:** ক্যারিয়ার গোল, টাইমলাইন, লার্নিং স্টাইল দিয়ে Gemini দিয়ে পার্সোনালাইজড রোডম্যাপ জেনারেট; DB-তে সেভ।

**Backend:** `POST /api/roadmap/generate` – Gemini, phases/milestones, MongoDB সেভ।  
**Frontend:** রোডম্যাপ জেনারেটর ফর্ম, প্রিভিউ, Save/View full roadmap।

---

## Feature 2: AI-Generated Skill Assessment Quizzes

**Problem:** পার্সোনালাইজড কুইজ নেই।

**Solution:** টপিক/মাইলস্টোন অনুযায়ী Gemini দিয়ে কুইজ জেনারেট; সাবমিটে ইভ্যালুয়েশন ও স্কোর সেভ।

**Backend:** `POST /api/quiz/generate`, `POST /api/quiz/submit` – জেনারেট + ইভ্যালুয়েশন + সেভ।  
**Frontend:** কুইজ পেজ (প্রশ্ন/অপশন), সাবমিট, রেজাল্ট পেজ (স্কোর, correct/incorrect)。

---

## Feature 3: Leaderboard & Points System

**Problem:** মোটিভেশন/র‍্যাংকিং নেই।

**Solution:** মাইলস্টোন/কোর্স/কুইজ থেকে পয়েন্ট; লিডারবোর্ড API ও পেজ।

**Backend:** পয়েন্ট রুল, প্রোগ্রেসে আপডেট; `GET /api/leaderboard`, `GET /api/user/points`।  
**Frontend:** লিডারবোর্ড পেজ (র‍্যাংক, নাম, পয়েন্ট), ইউজার নিজের র‍্যাংক হাইলাইট।

---

**চেকলিস্ট:** [ ] Feature 1  [ ] Feature 2  [ ] Feature 3  
**গ্লোবাল রিডমি:** [../README.md](../README.md)
