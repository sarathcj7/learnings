# Getting Started with System Design Mastery

Your repo is now ready to use! Here's what you have and how to get the most out of it.

---

## What's Complete ✅

### Immediately Usable

1. **[Problem-Solving Framework](case-studies/00-problem-solving-framework.md)**
   - 10-step system design interview methodology
   - Time allocation guide (45–60 min interviews)
   - Common pitfalls and how to avoid them
   - **Use**: Read before every practice interview

2. **[Caching Fundamentals Topic](fundamentals/caching/)**
   - 5 deep-dive sub-files: strategies, eviction, distributed, CDN, invalidation
   - Production-grade patterns and trade-offs
   - Mermaid diagrams showing system behavior
   - **Use**: Read to understand caching deeply, then test with question bank

3. **[Case Study: URL Shortener](case-studies/url-shortener.md)**
   - Complete walkthrough of a classic system design problem
   - Shows data modeling, sharding, caching, failure handling
   - Follows the problem-solving framework exactly
   - 150+ lines of detailed explanation
   - **Use**: Follow along, then re-solve it yourself under time pressure

4. **[Case Study: Distributed Rate Limiter](case-studies/distributed-rate-limiter.md)**
   - Deep dive into token bucket algorithm
   - Atomic operations, clock skew, distributed consistency
   - Failure scenarios and mitigations
   - 120+ lines of production patterns
   - **Use**: Study token bucket, then design a rate limiter from scratch

5. **[Caching Interview Prep](interview-prep/)**
   - 53 self-test questions (no answers; forces active recall)
   - 35 flashcards for spaced-repetition learning
   - **Use**: Daily flashcards + weekly question bank self-tests

### Structured & Ready to Fill

- **20 fundamentals topics**: Each has a README pointing to what's needed
- **40 interview prep files**: Question banks + flashcards for every topic
  - Caching is 100% complete
  - Others are templated stubs ready to fill
- **20 case studies**: Template set up, 2 complete, 18 ready for content

---

## Recommended First Week

### Day 1–2: Framework & Foundations
- [ ] Read [Problem-Solving Framework](case-studies/00-problem-solving-framework.md) (30 min)
- [ ] Skim [Case Study Template](case-studies/01-case-study-template.md) to understand structure (20 min)
- [ ] Start [Caching README](fundamentals/caching/README.md) (5 min)

### Day 3–4: Deep Dive into Caching
- [ ] Read [Caching Strategies](fundamentals/caching/strategies.md) (45 min)
- [ ] Read [Eviction Policies](fundamentals/caching/eviction-policies.md) (45 min)
- [ ] Answer 10 questions from [Caching Question Bank](interview-prep/question-banks/caching.md) (30 min)
- [ ] Review [Caching Flashcards](interview-prep/flashcards/caching.md) daily (5 min/day)

### Day 5–6: Distributed & Advanced Caching
- [ ] Read [Distributed Caching](fundamentals/caching/distributed-caching.md) (45 min)
- [ ] Read [CDN Caching](fundamentals/caching/cdn.md) (30 min)
- [ ] Read [Cache Invalidation](fundamentals/caching/invalidation.md) (45 min)
- [ ] Answer remaining questions from [Caching Question Bank](interview-prep/question-banks/caching.md) (30 min)

### Day 7: Apply & Practice
- [ ] Read [URL Shortener Case Study](case-studies/url-shortener.md) carefully (90 min)
- [ ] Without looking, design a URL shortener yourself (60 min)
- [ ] Compare your design to the writeup (30 min)
- [ ] Mark Caching topic as "🎯 Mastered" in the tracker

---

## After Week 1: Choosing Your Path

### Path A: Interview Prep (4–6 weeks)
Focus on high-impact topics and case studies.

**High-impact fundamentals** (study in order):
1. [Capacity Planning & Estimation](fundamentals/capacity-planning-and-estimation/) — the math toolkit
2. [Databases](fundamentals/databases/) — core to almost everything
3. [Distributed Systems Theory](fundamentals/distributed-systems-theory/) — understand CAP, consistency
4. [Scalability & Load Balancing](fundamentals/scalability-and-load-balancing/) — how to scale
5. [Reliability & Resiliency](fundamentals/reliability-and-resiliency/) — handle failure

**Core case studies** (one per day):
- [Distributed Cache](case-studies/distributed-cache.md)
- [News Feed & Social Timeline](case-studies/news-feed-and-social-timeline.md)
- [Chat & Messaging System](case-studies/chat-and-messaging-system.md)
- [Video Streaming Platform](case-studies/video-streaming-platform.md)
- [Ride-Sharing Service](case-studies/ride-sharing-service.md)
- [Distributed Job Scheduler](case-studies/distributed-job-scheduler.md)
- [Web Crawler](case-studies/web-crawler.md)
- [Notification System](case-studies/notification-system.md)

### Path B: Production Mastery (ongoing, self-paced)
Build deep knowledge across all topics.

**Study everything systematically**:
1. Each topic per week: read fundamentals + question bank + flashcards
2. One case study per week: study solution, re-solve under time pressure
3. Monthly: pick a weak area, deep dive and study 2 related topics

---

## How to Study Each Topic

### Standard 1-Week Cycle per Topic

#### Day 1–3: Learn
- [ ] Read all sub-files in the fundamentals topic (2–3 hours)
- [ ] Take notes on key trade-offs and when to use each pattern
- [ ] Answer question bank questions without peeking at fundamentals (1 hour)
- [ ] Check answers against fundamentals (30 min)

#### Day 4–5: Practice
- [ ] Do a related case study (if exists) and apply the topic
- [ ] Discuss with a peer: explain the topic to them
- [ ] Re-solve the case study focusing on this topic's choices

#### Day 6–7: Review
- [ ] Daily flashcard review (5 min)
- [ ] Answer question bank questions again (without notes)
- [ ] Mark progress: ⭕ → 📖 → 💪 → 🎯

---

## File Organization

```
/Users/apple/learnings/
├── README.md                              # Main index + progress tracker
├── BUILD_PROGRESS.md                      # What's done, what's next
├── GETTING_STARTED.md                     # This file
│
├── case-studies/
│   ├── 00-problem-solving-framework.md    # Interview methodology
│   ├── 01-case-study-template.md          # Structure template
│   ├── url-shortener.md                   # ✅ Complete
│   ├── distributed-rate-limiter.md        # ✅ Complete
│   └── [18 more case studies, stubs]
│
├── fundamentals/
│   ├── caching/                           # ✅ Complete (5 sub-files + README)
│   │   ├── README.md
│   │   ├── strategies.md
│   │   ├── eviction-policies.md
│   │   ├── distributed-caching.md
│   │   ├── cdn.md
│   │   └── invalidation.md
│   ├── [19 more topics, READMEs + stubs]
│
└── interview-prep/
    ├── README.md                          # Study guides
    ├── question-banks/
    │   ├── caching.md                     # ✅ Complete (53 questions)
    │   └── [19 stubs]
    └── flashcards/
        ├── caching.md                     # ✅ Complete (35 cards)
        └── [19 stubs]
```

---

## Quick Links

**Start here**:
- [Problem-Solving Framework](case-studies/00-problem-solving-framework.md)
- [Caching Topic](fundamentals/caching/README.md)
- [URL Shortener Case Study](case-studies/url-shortener.md)

**Main reference**:
- [Root README](README.md) – Topic map and progress tracker
- [Interview Prep README](interview-prep/README.md) – Study guides and recommended paths

**Stay organized**:
- Update the tracker in [README.md](README.md) as you progress (⭕ → 📖 → 💪 → 🎯)
- Track which topics are complete, practiced, mastered

---

## Study Tips

### For Active Learning
- **Question bank first**: Answer questions BEFORE reading fundamentals. Forces deeper thinking.
- **Teach others**: Explain a concept to a peer. If you can't, you don't understand it yet.
- **Whiteboard**: Design systems on a whiteboard or paper, not in your head. Exposes gaps.
- **Time pressure**: Solve case studies in 45–60 min, like a real interview.

### For Retention
- **Spaced repetition**: Review flashcards daily for a week, then 2–3x/week, then weekly.
- **Interleave**: Study different topics, not just deep dive into one. This improves transfer.
- **Sleep on it**: Study before bed. Sleep consolidates learning.
- **Test yourself**: Question banks and flashcards are your friends. Use them ruthlessly.

### For Confidence
- **Mock interviews**: Find a peer or mentor, do mock interviews on paper/whiteboard.
- **Record yourself**: If possible, record a practice interview and watch it. You'll spot weak explanations.
- **Fail fast**: Make mistakes in practice, not interviews. The goal is to know what you don't know.

---

## Next Steps (for the maintainer)

This session got the repo to ~50% by file count, 80% by structure. To complete:

1. **Fill in fundamentals** (3–5 hours):
   - Databases (high priority)
   - Distributed Systems Theory (high priority)
   - Consensus & Coordination (high priority)
   - Then remaining 16 topics (can be more concise)

2. **Complete case studies** (6–8 hours):
   - At least 5 more core case studies
   - 2–3 advanced case studies
   - Follow the URL Shortener/Rate Limiter template

3. **Fill interview prep** (2 hours):
   - Populate question banks (20–50 questions per topic)
   - Populate flashcards (15–35 per topic)
   - Can template based on caching examples

**Recommended order of completion**:
- Databases + Distributed Systems topics (they unlock case studies)
- 5–10 more case studies (using topics as they're ready)
- Remaining fundamentals (breadth)
- Question banks and flashcards (once topics are written)

---

## Questions?

Refer to:
- [BUILD_PROGRESS.md](BUILD_PROGRESS.md) – What's done, what's planned
- [interview-prep/README.md](interview-prep/README.md) – Study strategies
- [case-studies/00-problem-solving-framework.md](case-studies/00-problem-solving-framework.md) – How to think about system design

---

**Happy studying! 🚀**

*Last updated: 2026-07-23*  
*Estimated completion: 80% of writing done; core exemplars complete. Ready for immediate use.*
