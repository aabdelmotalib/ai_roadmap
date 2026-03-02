# Your Study Planner: How to Actually Finish This Roadmap

Most people don't finish learning roadmaps.

Not because they're not smart enough. Because they don't have a schedule.

This guide gives you the structure to actually finish. For real.

---

## 1. Weekly Schedule Templates

### Full-Time Learner (40 hours/week)

You're doing this full-time. School break, sabbatical, job transition period.

| Day | Morning (3h) | Afternoon (3h) | Evening (2h) | Evening (2h) |
|-----|---|---|---|---|
| **Monday** | New topic: read slides/docs (Phase) | Code examples from topic | Build project code | Review + flashcards |
| **Tuesday** | Continue topic (deeper dive) | Code examples + debugging | Build project + test fixes | Reflection: what's confusing? |
| **Wednesday** | New topic (if ready) or deep dive | Code implementation | Project build (intensive) | Document what you learned |
| **Thursday** | Review previous topics | Practice old concepts | Project build | Quiz yourself (can you explain?) |
| **Friday** | New topic or catch-up | Code exploration | Project shipping | Weekend prep |
| **Saturday** | Project refinement | Advanced features | Optimization / polish | Blog post writing |
| **Sunday** | Recovery + review | Catch-up on weak areas | Light reading ahead | Plan next week |

**Weekly targets:**
- 12 hours: reading/learning
- 16 hours: coding/building
- 8 hours: review/reflection
- 4 hours: stretch goals

This pace gets you through the full roadmap in 20-24 weeks.

### Part-Time Learner (15 hours/week)

You're working full-time or have limited time. This path prioritizes building over reading.

**Best approach: weekday evenings + weekend blocks**

**Weekday (10 hours/week, concentrated):**
- Monday-Tuesday: New topic (2h), code examples (1h/day) = 4 hours
- Wednesday: Deep dive coding = 2 hours
- Thursday-Friday: Practice + project = 4 hours

**Weekend (5 hours):**
- Saturday: 3-4 hour focused building session
- Sunday: 1.5 hour review + next week prep

**Skip:** Optimization, polish, advanced features. Focus on core understanding and working code.

**Realistic timeline:** 30-36 weeks (7-9 months)

### Weekend Warrior (8 hours/week)

You have a full-time job. You learn on weekends only.

**Saturday morning (5h): Deep focus**
- No notifications. Phone on silent.
- 1.5h: Read/learn one concept
- 3h: Code that concept
- 0.5h: Test and verify

**Sunday evening (3h): Reinforce + plan**
- 1.5h: Code review what you built
- 1h: Test and document
- 0.5h: Plan next weekend's topic

**Constraint:** You won't do advanced projects. You'll understand the important concepts and build working code, but not polished systems.

**Realistic timeline:** 40-50 weeks (9-12 months)

**Pro tip:** Use your weekday commute (30 min) for spaced review. Flashcards, re-watching concepts. This helps retention without eating into learning time.

---

## 2. The 70/30 Rule

**70% building. 30% reading.**

Not the other way around.

### How to Enforce This

Create a simple spreadsheet:

| Week | Phase | Study Hours | Build Hours | Ratio | On Target? |
|------|-------|---|---|---|---|
| 1 | Intro | 2 | 8 | 80% build ✓ | Yes |
| 2 | Embeddings | 4 | 10 | 71% build ✓ | Yes |
| 3 | Embeddings | 4.5 | 6 | 57% build ✗ | Too much reading |
| 4 | RAG | 3 | 12 | 80% build ✓ | Yes |

Track actual time. Adjust.

### Signs You're Breaking the Rule (and how to fix it)

!!! warning "Sign: You're reading more than coding"
    Problem: You're in tutorial hell. Reading is comfortable because you don't fail.
    
    Fix: Set a timer. 30 min reading max per topic, then code for 2 hours.

!!! warning "Sign: You haven't shipped anything in 2 weeks"
    Problem: Too much setup/learning, not enough building.
    
    Fix: Today, make something work. Doesn't have to be perfect. Just functional.

!!! warning "Sign: You're switching topics every 2-3 days"
    Problem: You're overwhelmed or jumping around.
    
    Fix: Stick with one topic for 2-3 days minimum. Build something complete.

!!! warning "Sign: You understand it intellectually but can't code it"
    Problem: You're learning without building.
    
    Fix: Write code before reading the explanation. Struggle first. Then read.

---

## 3. When You're Stuck: The Unstuck Ladder

You will get stuck. Everyone does.

Follow this in order. Do each for 10 minutes before moving to the next.

### Level 1: Rubber Duck (10 min)

Explain the problem out loud to a rubber duck (or your pet, or yourself).

"My RAG retrieval returns wrong results. I want it to return the most similar chunk, but it returns irrelevant chunks. When I embed the query and search, the similarity scores are 0.6-0.7, but 0.85 threshold returns nothing. So either 1) embedding is wrong, 2) similarity threshold is wrong, 3) data is bad..."

Talking forces clarity. You often find the bug while explaining.

### Level 2: Re-read the Error (10 min)

Your instinct is to Google the error.

Instead: read it slowly.

"ValueError: `shape (100,) and (384,) are not compatible`"

Translation: You're trying to compare a 100-dimensional vector to 384-dimensional vector. They're different dimensions.

Why? Your embedding model changed, or you're loading wrong model.

### Level 3: Check the Debugging Playbook (10 min)

Each lecture file has a "Debugging" section.

[RAG Debugging Playbook](../phase1/rag.md#debugging) has 10 common errors. Is yours there?

### Level 4: Ask Claude with Full Context (10 min)

Now you can ask effectively:

```
Error: similarity returns value outside [-1, 1]
Got: 1.05

Code:
[full relevant code]

What I've tried:
- Checked embedding dimensions match (they do, both 384)
- Checked similarity function (using cosine_similarity from sklearn)
- Input embeddings are normalized (using l2)

My hypothesis:
Floating point precision? Cosine should clamp to [-1,1].

What am I missing?
```

Claude will likely say: "1.05 due to floating point precision, totally normal. Add `.clip(-1, 1)` to be safe."

You just learned something real.

### Level 5: Better Searches (10 min)

Bad search: "vector similarity"
Good search: "sklearn cosine_similarity returns > 1"

Include library, version, exact error. 

Likely finds stack overflow with exact answer.

### Level 6: Ask in Community (30 min)

Try [HuggingFace Forums](https://discuss.huggingface.co), [r/LanguageModels](https://reddit.com/r/LanguageModels), [FastAPI GitHub Discussions](https://github.com/tiangolo/fastapi/discussions).

Provide:
- Minimal reproducible example
- What you've tried
- Your hypothesis

Real humans come through. Takes a few hours, but you get clarity.

### Level 7: Sleep on It (24h)

Genuine rest is a underrated debugging tool.

Take a walk. Shower. Sleep.

Your subconscious keeps working. You come back with a fresh perspective. The solution is often obvious the next day.

---

### What To Do When You're Stuck for 2+ Hours

**Stop. Seriously.**

You're in a negative spiral. The stress is blocking problem-solving.

Time-box and move on:

```
11:00 - Run into problem
11:00-11:10 - Rubber duck
11:10-11:20 - Read error again
11:20-11:30 - Check debugging guide
11:30-11:40 - Ask Claude
11:40-12:00 - Searches
12:00 - SWITCH TASKS

Come back tomorrow with fresh mind.
```

You'll often fix it in 5 minutes the next day.

---

## 4. Progress Tracking System

You don't need a fancy app. A text file works perfectly.

### Plain Text Progress Tracker

Create: `PROGRESS.md`

```markdown
# Learning Progress

## Phase 0: Foundation
- [x] Python Mastery
  - [x] Read all sections
  - [x] Completed code examples
  - [x] Built sample project
  - [x] Passed review (8/10)
- [ ] Git & GitHub
  - [x] Read sections
  - [x] Code examples
  - [ ] Create GitHub account and practice
  - [ ] Review material (target: 8/10)

## Phase 1: AI Systems
- [ ] LLM Fundamentals
  - [ ] Read all sections
  - [ ] Completed code examples
  - [ ] Built sample project
  - [ ] Passed review (8/10)
- [ ] Embeddings
  - [x] Read all sections
  - [x] Completed code examples
  - [ ] Built sample project (IN PROGRESS - 50% done)
  - [ ] Passed review (target: 8/10)

...

## Metrics
- Total phases: 6
- Phases started: 2
- Phases completed: 1 (Phase 0)
- Current phase: Phase 1 (30% done)
- Total hours invested: 45 hours
- Weeks into roadmap: 4 weeks
- Pace: 11h/week (on track for 20-week completion)
```

Update it weekly. It's your progress mirror.

### When to Mark a Topic Complete?

Not "I read it." But actually "I can use it."

✅ **Can explain it without notes** (even rough explanation)
✅ **Can write the code from memory** (not line-for-line, just the structure)
✅ **Passed 8/10 review questions** (each topic has self-test questions)
✅ **Built the practical project** (not polished, but functional)

If you can do all 4: complete.

---

## 5. Ready to Move On vs. Rushing

### Signs You're Ready for Next Topic ✅

- You can explain the concept in your own words (to a friend, or in writing)
- You built working code without copy-pasting the solution
- You understand why the code works, not just what it does
- You can predict what happens if you change one line
- You've spent 2-3 days with the topic (not rushed through in 3 hours)

### Signs You're Rushing (Not Ready) ❌

- You haven't built the practical project yet
- Code works because you copied the solution, not writing it
- You can't explain why something works (just that it does)
- You'll need to re-learn it in 2 weeks when you forget
- You're jumping topics because they're hard or slow
- You can't answer 6 of the 10 review questions

**If you see rushing signs: STOP. Go back. Spend 2-3 more days.**

You'll save time in the long run because you won't need to re-learn.

---

## 6. Handling Low-Motivation Weeks

This will happen. Usually around week 6-8 when novelty wears off.

### The 5-Minute Rule

Don't ask yourself "Do I want to study?"

Ask: "Can I spend 5 minutes on my laptop opening my project?"

Just 5 minutes. No commitment.

99% of the time, you open, see your code, remember why it's interesting, and keep working.

### Momentum Techniques

**Look at what you've built:**
- Scroll through your GitHub commits
- Run your projects
- See them working
- Remember you've made real progress

Don't compare to what you haven't built yet. That kills motivation.

**Make one small thing work:**
- Not "finish the whole phase"
- Just "make the button work" or "fix this one bug"
- Completion is motivating, no matter how small

**Community:** Tell someone what you're building.
- Tweet about your progress (you'd be surprised how supportive AI engineers are)
- Show a friend
- Post on Discord

External accountability is a powerful motivator.

### What NOT to Do

!!! warning "Don't: Take a week off"
    A week off becomes two weeks. Two weeks becomes a month.
    
    You lose momentum and restarting is hard.

!!! warning "Don't: Switch to a different learning path"
    "Maybe I should learn JavaScript instead of AI..."
    
    This is your brain's way of avoiding hard topics. Push through.

### How to Reduce Without Stopping

Low motivation week? Cut to 50% your normal load.

```
Normal: 10h study, 15h build per week
Low week: 5h study, 7.5h build per week

Still moving forward, just slower. That's fine.
```

Don't pause entirely. Just 50%.

---

## 7. Learn in Public: Accelerate Through Sharing

Learning alone is slow. Learning in public is 3x faster.

Why? Explaining forces clarity. Feedback corrects mistakes. Community celebrates wins.

### Weekly Content Ideas

**Monday Twitter/X thread:**
```
This week I learned about semantic caching. Here's 3 things that surprised me:

1/ Caching embeddings is 30x cheaper than recomputing. 
   cos_sim(query, cached) > 0.95 and return previous answer
   Saved $500/month on inference.

2/ Thread-safety matters. Two requests same query = race condition.
   Redis SET with locks prevents double-compute.

3/ Cold start is your friend. Cache misses on cold start teach you 
   which queries users care about.

Next week tackling batch inference. GitHub: [link] 💻
```

**Frequency:** Once per week, 2-4 minutes to write, reaches 100-1000 people

**LinkedIn post (bi-weekly):**
```
Built a semantic caching layer today and 3 lessons stuck with me:

1. Sometimes the highest ROI optimization is the simplest.
2. Measure before and after. We cut latency by 45%.
3. Your database doesn't always need to be your bottleneck.

Full repo: [link to GitHub]
```

**GitHub:** Consistent commits visible on your contribution graph
```
commit a3f34k "feat: add semantic caching with Redis backend"
commit b2d45l "test: add caching layer unit tests"
commit c3e56m "docs: update README with caching benchmarks"
```

Visible progress attracts opportunities.

**Blog post (once per phase):**
```
"Building RAG: What I Learned"
- What problem I was solving
- Architecture I built
- 5 things that surprised me
- Code snippets + GitHub link
- What's next

5 min read, 500-1000 words. Worth your time.
```

### Template: Weekly Twitter/X Thread

```
This week I learned about [TOPIC]. Here's 3 things that surprised me:

1/ [Specific insight 1: something you didn't expect]
   [Why it matters: practical impact]

2/ [Specific insight 2: concrete lesson]
   [Why it matters: problem it solves]

3/ [Specific insight 3: learning point]
   [Why it matters: how you'll use it]

Building [WHAT]: [GitHub link]
Next week: [NEXT TOPIC]
```

Specific > Generic
Concrete > Abstract
Your voice > ChatGPT voice

---

## 8. Phase-by-Phase Time Estimates

Realistic estimates (not optimistic):

| Phase | Topic | Full-Time (40h/w) | Part-Time (15h/w) | Weekend (8h/w) |
|-------|-------|---|---|---|
| 0 | Python & Git | 1 week | 2-3 weeks | 4-5 weeks |
| 1 | AI Systems | 5 weeks | 10-12 weeks | 15-18 weeks |
| 2 | Deployment | 3 weeks | 6-8 weeks | 10-12 weeks |
| 3 | MLOps | 3 weeks | 6-8 weeks | 10-12 weeks |
| 4 | Kubernetes | 4 weeks | 8-10 weeks | 12-15 weeks |
| 5 | Jobs | 2 weeks | 4-5 weeks | 6-8 weeks |
| **Total** | **6 phases** | **18-20 weeks** | **36-48 weeks** | **57-70 weeks** |

!!! note "Real Talk About Time Estimates"
    These are if you're:
    - Not starting from zero (know Python basics)
    - Focus and consistent
    - No major life disruptions
    
    Reality: add 20-30% buffer for life happening.

### Adjusting If You're Behind

You're 6 weeks in, only done 1.5 phases (should be 2).

Options:
1. **Speed up slightly** (10h/week → 12h/week)
2. **Skip optional deep-dives** (implement, don't optimize)
3. **Extend timeline** (it's fine, learning > rushing)
4. **Combine phases** (do MLOps + K8s together since they overlap)

Most people choose option 3. That's wise.

---

## Summary: Your Study Rhythm

**Pick your schedule:**
- Full-time: 20 weeks
- Part-time: 36-48 weeks
- Weekend: 57-70 weeks

**Weekly ritual:**
- Monday: Start new topic
- Tuesday-Thursday: Build and practice
- Friday: Polish, review
- Weekend: Reflect and plan next week

**70/30 rule:** 70% coding, 30% reading

**Stuck?** Unstuck ladder (rubber duck → searches → sleep)

**Tracking:** One markdown file, updated weekly

**Motivation:** Low weeks are 50%, not zero. Learn in public. Celebrate small wins.

**Ready?** Start slowly. You'll find your pace.

---

## Your First Week Checklist

- [ ] Pick your schedule (full-time, part-time, or weekend warrior)
- [ ] Create PROGRESS.md file
- [ ] Block time on your calendar (same time each week)
- [ ] Set up notifications (gentle reminders, not aggressive)
- [ ] Join a community (Discord, Reddit, or AI engineers community)
- [ ] Commit to one week (7 days) before deciding if schedule fits
- [ ] Track actual hours to validate your pace assumption
- [ ] Plan your first project (Phase 0)
- [ ] Share your goal with one person (accountability buddy)

---

**Next:** [Project Templates →](project-template.md)
