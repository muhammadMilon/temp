# Muhammad Milon – Task Details (Team Leader)

**Focus:** Complex logic + AI-related tasks  
**Minimum:** ৩টি কমপ্লিট মেজর ফিচার (Frontend + Backend)

---

## Feature 1: AI Skill Gap Analysis

**Problem:** লার্নাররা জানে না কারেন্ট স্কিল ও টার্গেট ক্যারিয়ারের মধ্যে কী গ্যাপ আছে।

**Solution:** ফর্ম ইনপুট (কারেন্ট স্কিল, টার্গেট রোল) নিয়ে Gemini API দিয়ে অ্যানালাইসিস; রেজাল্টে existingSkills ও gapSkills দেখানো।

**Backend:** `POST /api/skill-gap/analyze` – Gemini কল, রেজাল্ট সেভ (optional)।  
**Frontend:** স্কিল গ্যাপ ফর্ম, লোডিং/এরর, রেজাল্ট সেকশন ও “Generate Roadmap” লিংক।

---

## Feature 2: Adaptive Roadmap Updates (AI)

**Problem:** রোডম্যাপ প্রোগ্রেস/কুইজ অনুযায়ী আপডেট হয় না।

**Solution:** প্রোগ্রেস ও কুইজ ডেটা দিয়ে Gemini দিয়ে আপডেটেড মাইলস্টোন জেনারেট; DB আপডেট।

**Backend:** `POST /api/roadmap/adapt` – প্রোগ্রেস/কুইজ অ্যাগ্রিগেট, Gemini, DB আপডেট।  
**Frontend:** ড্যাশবোর্ড/রোডম্যাপে “AI updated your path” ব্লক ও অ্যাপ্লাই/ডিসমিস।

---

## Feature 3: AI Feedback Loop (Rate → Adapt)

**Problem:** ইউজার রেটিং সিস্টেম ব্যবহার করে না, রিকমেন্ডেশন পার্সোনালাইজ হয় না।

**Solution:** মাইলস্টোন/কোর্স রেটিং সেভ; ফিডব্যাক দিয়ে পরবর্তী রিকমেন্ডেশন অ্যাডজাস্ট।

**Backend:** `POST /api/feedback/milestone`, `POST /api/feedback/course`; ফিডব্যাক রিকমেন্ডেশন লজিকে ব্যবহার।  
**Frontend:** মাইলস্টোন/কোর্স পেজে রেটিং UI (stars/thumbs), সাবমিট ও সাকসেস মেসেজ।

---

**চেকলিস্ট:** [ ] Feature 1  [ ] Feature 2  [ ] Feature 3  
**গ্লোবাল রিডমি:** [../README.md](../README.md)
