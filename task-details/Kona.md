# Kona – Task Details

**Focus:** AI-related features  
**Minimum:** ৩টি কমপ্লিট মেজর ফিচার (Frontend + Backend, Problem → Solution)

---

## Feature 1: AI-Based Course Recommendations

**Problem:** হাজারো কোর্স থাকলে ইউজার জানে না তার গোল ও কারেন্ট স্কিল অনুযায়ী কী কী কোর্স নেওয়া উচিত।

**Solution:** ইউজার রোডম্যাপ/গোল ও স্কিল ডেটা দিয়ে Gemini API দিয়ে পার্সোনালাইজড কোর্স রিকমেন্ডেশন লিস্ট জেনারেট করা (Coursera, Udemy, YouTube ইত্যাদি সোর্স মেনশন করে)।

**Backend:**
- API: `POST /api/recommendations/courses` বা `GET /api/recommendations/courses` (userId বা roadmapId, limit)
- ইউজার প্রোফাইল/রোডম্যাপ/স্কিল গ্যাপ ডেটা অ্যাগ্রিগেট করা
- Gemini API কে context দিয়ে কোর্স রিকমেন্ড জেনারেট (নাম, লিংক, সংক্ষিপ্ত কারণ)
- রেজাল্ট ক্যাশ বা DB-তে সেভ করে পরবর্তী রিকোয়েস্টে ব্যবহার (optional)

**Frontend:**
- ড্যাশবোর্ডে “AI-Recommended Courses” সেকশন
- কোর্স কার্ড: টাইটেল, সংক্ষিপ্ত বর্ণনা, “View course” লিংক
- লোডিং ও এরর স্টেট

**Deliverables:** Recommendation API, Gemini integration, Dashboard UI for recommended courses।

---

## Feature 2: Quiz Results → Roadmap Adjustment Suggestions

**Problem:** কুইজ দেয়ার পর রেজাল্ট শুধু স্কোর দেখায়; রোডম্যাপে সেই অনুযায়ী কী শিখতে হবে বা কী রিভাইজ করতে হবে সেটা সুজেস্ট হয় না।

**Solution:** কুইজ রেজাল্ট (টপিক, স্কোর, ভুল অ্যান্সার) দিয়ে AI দিয়ে “এই টপিকগুলো রিভাইজ করো” বা “পরবর্তী মাইলস্টোনগুলো এভাবে নেও” টাইপের সাজেশন জেনারেট করা।

**Backend:**
- কুইজ সাবমিশন ও রেজাল্ট ডেটা থেকে weak topics / strong topics বের করা
- API: e.g. `POST /api/roadmap/suggestions-from-quiz` (quizResultId বা userId + recentQuizIds)
- Gemini কে রেজাল্ট সামারি দিয়ে টেক্সট সাজেশন জেনারেট (রিভাইজ লিস্ট, নেক্সট স্টেপস)
- রেসপন্স সেভ করে ফ্রন্টএন্ডে দেখানো

**Frontend:**
- কুইজ রেজাল্ট পেজে “AI suggestions” ব্লক: “Revise these topics”, “Suggested next steps”
- (Optional) “Update my roadmap” বাটন দিয়ে অ্যাডাপ্ট রোডম্যাপ API কলে দেয়া (Milon এর ফিচার সাথে ইন্টিগ্রেট)

**Deliverables:** Suggestion API using quiz results + Gemini, UI on quiz results page।

---

## Feature 3: Trending Skills & Roadmaps (Curated + Analytics)

**Problem:** ইউজাররা জানতে চায় ইন্ডাস্ট্রিতে কোন স্কিল বা রোডম্যাপ ট্রেন্ড করছে; স্ট্যাটিক লিস্ট যথেষ্ট না।

**Solution:** অ্যাপের ইউজার ডেটা (কতজন কোন রোডম্যাপ/স্কিল এ এনরোল করছে, কমপ্লিশন রেট) অ্যাগ্রিগেট করে ট্রেন্ডিং লিস্ট বের করা; অপশনালি AI দিয়ে সংক্ষিপ্ত “কেন ট্রেন্ডিং” ডিসক্রিপশন জেনারেট করা।

**Backend:**
- অ্যাগ্রিগেশন: রোডম্যাপ/স্কিল অনুযায়ী এনরোলমেন্ট, কমপ্লিশন কাউন্ট
- API: `GET /api/trending/skills`, `GET /api/trending/roadmaps` (limit, timeRange optional)
- (Optional) Gemini দিয়ে প্রতিটি ট্রেন্ডের জন্য এক লাইন বর্ণনা জেনারেট

**Frontend:**
- ট্রেন্ডিং স্কিলস পেজ: কার্ড/লিস্ট (স্কিল নাম, ডিমান্ড ইন্ডিকেটর বা %, সংক্ষিপ্ত বর্ণনা)
- ট্রেন্ডিং রোডম্যাপস (optional): পপুলার রোডম্যাপ লিস্ট ও “Explore” লিংক

**Deliverables:** Aggregation logic, trending APIs, trending skills (ও optional roadmaps) UI।

---

## টাস্ক চেকলিস্ট

- [ ] Feature 1: AI Course Recommendations – Backend + Gemini + Dashboard UI
- [ ] Feature 2: Quiz → Roadmap suggestions – Backend suggestions API + Quiz results page UI
- [ ] Feature 3: Trending Skills (ও optional Roadmaps) – Backend analytics + Frontend page

**গ্লোবাল রিডমি:** [../README.md](../README.md)
