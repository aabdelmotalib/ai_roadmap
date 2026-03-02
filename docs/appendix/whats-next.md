# What's Next: Your Path After This Roadmap

You've finished the roadmap. Congratulations.

But it's not an ending. It's a beginning.

This guide shows you where to go next based on what excites you.

---

## 1. Specialization Paths

The roadmap made you a full-stack AI engineer. That's rare and valuable.

Now you choose depth.

### Path 1: AI Research Engineer

**Focuses on:** Fine-tuning at scale, RLHF, evaluation frameworks, pushing model capabilities, novel architectures

**Why choose this:** You love the science. You want to build the models, not just use them. You're excited by papers, training runs lasting weeks, optimizing for marginal improvements.

**This is for you if:**
- You think about model architecture improvements
- You run ablation studies (swapping components to understand impact)
- You read papers for fun
- You care about theoretical understanding, not just results

**Next Skills to Build:**
- **RLHF (Reinforcement Learning from Human Feedback):** How models learn to be helpful
  - Library: TRL (Transformers Reinforcement Learning)
  - Concept: Train a reward model, use it to improve base model
  - Cost: Expensive (need GPU cluster), months to train
- **Constitutional AI:** Aligning models without human feedback
  - Concept: Give models a constitution ("be helpful, harmless, honest"), they self-improve
  - Companies: Anthropic is leading this
- **Evaluation frameworks:** How do you actually measure model quality?
  - RAGAS (for RAG)
  - HELM (for foundation models)
  - Custom evaluators for your domain
- **Distributed training:** Training models across multiple GPUs/TPUs
  - Libraries: Hugging Face Transformers, DeepSpeed, Megatron
  - Concept: Split model or data across machines
- **Reading and implementing research papers:** Weekly ritual of understanding papers, building them
  - Start with: arxiv.org → filter by "language models"
  - Implement: Pick papers, code them, publish on GitHub

**Time to Senior Level:** 3-4 years (research moves fast, you learn by doing)

**Salary ceiling:** $200K-$300K+ (senior researcher at major labs)

**Companies hiring:** Anthropic, OpenAI, Meta AI, Cohere, Mistral, Google DeepMind, Hugging Face

**Reality check:** This path requires:
- Comfort with uncertainty (experiments fail)
- Patience (training runs take weeks)
- Math background (helpful but not essential with libraries)
- Love of iteration (many small experiments)

---

### Path 2: MLOps / AI Infrastructure Engineer

**Focuses on:** Kubernetes at scale, data pipelines, model serving optimization, feature stores, model monitoring, infrastructure for production ML

**Why choose this:** You love solving the "scale" problem. You Kubernetes. You want models running reliably at millions of requests/second. You optimize infrastructure, not algorithms.

**This is for you if:**
- You ask "will this scale to 10M users?" when seeing a design
- You love infrastructure problems (load balancing, caching, failover)
- You're excited by monitoring and alerting
- You prefer system design over algorithm design

**Next Skills to Build:**
- **Kubeflow:** ML workflows on Kubernetes
  - Concept: Pipelines for training, serving, monitoring
  - Components: Katib (hyperparameter tuning), KServe (model serving), Pipelines
  - When: 1-2 weeks to get comfortable
- **Airflow / Prefect:** Data pipeline orchestration
  - Concept: Schedule and track data processing workflows
  - Use case: Daily model retraining, data processing jobs, ETL
- **Ray:** Distributed computing for ML
  - Concept: Parallelization for training and inference
  - Speed: 10-100x faster than single machine
- **Feature Stores (Feast, Tecton):** Manage features at scale
  - Concept: Centralized place for feature definitions, serving, monitoring
  - Problem it solves: Feature consistency between training and serving (critical bug source)
- **Model Serving Optimization:**
  - vLLM: Continuous batching for LLMs (faster inference with better GPU utilization)
  - TensorRT-LLM: NVIDIA's optimized inference
  - BentoML: Model serving framework
- **Monitoring and observability:** Beyond basic metrics
  - Prometheus + Grafana: Metrics
  - ELK / Datadog: Logging
  - Jaeger / DataDog: Tracing
  - Custom model performance dashboards

**Time to Senior Level:** 2-3 years (infrastructure is well-understood, you're building on proven patterns)

**Salary ceiling:** $250K-$350K+ (senior infra engineer)

**Companies hiring:** All large tech companies, MLOps startups like Anyscale, Databricks, Platform.io, Verta, Weights & Biases

**Reality check:** This path requires:
- Love of operations (not everyone does)
- Patience with outages (infrastructure breaks, you fix it)
- Continuous learning (tools change fast, but principles are stable)
- On-call comfort (infrastructure support often has rotation)

---

### Path 3: AI Product Engineer (Full Stack + AI)

**Focuses on:** Product thinking + AI features, UX for AI, latency optimization, full-stack development, feature velocity

**Why choose this:** You love shipping features users love. You want to add AI to products, not build AI itself. You optimize for user happiness, not model accuracy.

**This is for you if:**
- You ask "will users actually want this?" for every feature
- You care about latency as much as accuracy
- You love full-stack work (frontend + backend + AI)
- You get excited by shipping fast and iterating

**Next Skills to Build:**
- **Real-time response systems:**
  - WebSockets for streaming responses
  - Server-sent events (SSE) for streaming from LLMs
  - Latency targets: 100ms for UI responsiveness
  - Concept: Stream token-by-token, not wait for full response
- **Multimodal AI:** Vision + text + audio
  - GPT-4V / Claude Vision: Image understanding
  - Whisper: Speech to text
  - Texture: Text to speech
  - Use case: Upload screenshot, AI explains what's on screen
- **A/B Testing AI Features:** How do you measure success?
  - Metrics beyond accuracy: user engagement, retention, satisfaction
  - Framework: Hypothesis → test → measure → iterate
- **User research for AI:** How do users want to interact with AI?
  - Talk to users about AI features
  - Watch how they use it (not how they say they use it)
- **Cost optimization for users:** Cheaper models, caching, local inference
  - When to use GPT-3.5 (faster, cheaper) vs GPT-4
  - When to run locally (Ollama, llama.cpp) vs API
  - Caching to reduce API calls (semantic caching pays off here)
- **Agentic frameworks:** Teaching AI to do multi-step tasks
  - LangGraph: Building agent workflows
  - AutoGen: Multi-agent orchestration
  - Concept: AI makes decisions, takes actions, learns

**Time to Senior Level:** 2-3 years (product sense develops through shipping)

**Salary ceiling:** $300K-$400K+ (senior product engineer at successful AI companies)

**Companies hiring:** AI-first startups (Vercel, Cursor, Replit, Anthropic's Claude API customers), big tech adding AI features (Apple, Google, Meta, Microsoft)

**Reality check:** This path requires:
- User empathy (you must care about users, not just code)
- Comfort with ambiguity (product direction isn't always clear)
- Speed over perfection (ship, learn, iterate)
- Balance (sometimes good-enough is better than perfect)

---

## 2. Advanced Topics Roadmap

Want to stay current? These topics define the cutting edge of AI engineering.

| Topic | What It Is | Why It Matters | Entry Point | Time to Competency | Companies Doing This |
|-------|-----------|---|---|---|---|
| **Multimodal AI** | Vision + text + audio in one model | Why text-only is limiting; images/audio have context | GPT-4V documentation | 1-2 weeks | OpenAI, Anthropic, Deepseek |
| **Voice AI** | Speech recognition + synthesis + conversation | Voice interfaces are the future UX | Whisper (OpenAI) + ElevenLabs API | 2-3 weeks | OpenAI, ElevenLabs, Google, Apple |
| **LLM Inference Optimization** | Making inference faster and cheaper | Models are getting bigger, inference is the bottleneck | vLLM documentation | 3-4 weeks | vLLM (UC Berkeley), NVIDIA |
| **Quantization** | Smaller, faster models via fewer bits | 7B parameters at 4-bit = same quality, 4x smaller | GPTQ, AWQ papers + libraries | 2-3 weeks | NVIDIA, Hugging Face, Together.ai |
| **Advanced Fine-tuning** | RLHF, DPO, reward models | Getting models to behave how you want | TRL library + Constitutional AI | 4-6 weeks | Anthropic, OpenAI, Hugging Face |
| **Agentic Frameworks** | Multi-step AI reasoning and planning | Agents are the next level beyond chatbots | LangGraph, AutoGen docs | 3-4 weeks | Anthropic, OpenAI, LangChain |
| **Data Infrastructure** | Pipelines, feature stores, data lakes | Production systems need reliable data | Feast, Tecton, dbt | 4-6 weeks | Tecton, Databricks, dbt Labs |
| **Retrieval Augmentation Advanced** | Hybrid search, re-weighting, multi-hop retrieval | Basic RAG is insufficient for complex reasoning | LlamaIndex technical papers | 3-4 weeks | LlamaIndex, Weaviate |
| **Model Alignment & Safety** | Constitutional AI, red teaming | AI safety is increasingly important and lucrative | Anthropic's papers, MIT course | 6-8 weeks | Anthropic, Alignment Research Center |
| **Edge AI & ONNX** | Running models on devices (phones, IoT) | Privacy + latency by keeping models local | ONNX standards, TensorFlow Lite | 3-4 weeks | Apple, Google, Qualcomm |

---

### Recommended Learning Order

**Month 1-2:** Multimodal AI + Voice AI (high impact, accessible)
**Month 3:** LLM Inference Optimization (immediately useful)
**Month 4-5:** Advanced fine-tuning + Quantization (depth)
**Month 6+:** Agentic frameworks + Data infrastructure (scale)

---

## 3. Communities to Join

Learning in isolation is possible. Learning in community is 3x faster.

### Hugging Face Discord

**What:** 50K+ researchers, engineers, students
**Best for:** Open-source model questions, dataset discussions, competition participation
**How to join:** [huggingface.co/discord](https://huggingface.co/discord)
**Activity level:** 100+ messages/day
**Quality:** Very high (moderated, expert advice)

**How to get value:**
- Post your question with context (error message, code, what you've tried)
- Follow #announcements for new models and competitions
- Observe #papers channel for research discussions
- Help others (teaching solidifies your own knowledge)

---

### LangChain Discord

**What:** 100K+ developers using LLMs
**Best for:** RAG, agents, prompt engineering, LangChain library support
**How to join:** [discord.gg/langchain](https://discord.gg/langchain)
**Activity level:** 200+ messages/day
**Quality:** High, friendly community

**What happens here:**
- Someone posts a RAG problem, 5 people suggest solutions
- New LLM API released, someone shows how to use it with LangChain
- You share your project, get feedback

---

### MLOps Community Slack

**What:** 50K+ MLOps engineers
**Best for:** Production ML systems, infra questions, hiring
**How to join:** [mlops.community](https://mlops.community)
**Activity level:** 50+ messages/day in each channel
**Quality:** Very professional, people solving real problems

**Channels worth following:**
- #general: announcements and meta discussions
- #k8s-for-ml: Kubernetes for ML
- #feature-stores: Feature store discussions
- #jobs: Recruiting engineers

---

### LocalLLaMA (Reddit)

**What:** 300K+ people running LLMs locally
**Best for:** Fine-tuning, quantization, local models
**How to join:** [reddit.com/r/LocalLLaMA](https://reddit.com/r/LocalLLaMA)
**Activity level:** 50+ new posts/day
**Quality:** Mixed, but mods are good

**Best posts:**
- New model releases (Mistral, Llama, Openhermes)
- Benchmarks (which model is best for task X?)
- How-tos for running models

---

### AIEngineers (aie.fyi)

**What:** 1K+ AI engineers in Slack
**Best for:** Practitioner network, job referrals, real-world problems
**How to join:** [aie.fyi](https://aie.fyi)
**Activity level:** 20+ messages/day
**Quality:** Excellent (vetted members, experienced engineers)

**Ask here:**
- "I'm building X, encountered Y, tried Z, now stuck"
- "Which tool should I use for problem A?"
- Job referrals and opportunities

---

### Latent Space Community

**What:** Practitioner podcast + Discord
**Best for:** Staying current with AI landscape, interesting interviews
**How to join:** [latentspace.ai/discord](https://latentspace.ai)
**Activity level:** Discussion of podcast episodes, 20+ messages/day
**Quality:** Very high, smart engineers

**Bonus:** Subscribe to the podcast (weekly 90 minutes, but skip intros)

---

## 4. Newsletters and Podcasts

Stay current with AI without getting overwhelmed.

### Must-Read Newsletters

**The Batch (Andrew Ng)**
- Frequency: Weekly
- Time to read: 10 minutes
- Content: AI research summaries, application spotlights
- Why: Authoritative, educational, accessible
- Subscribe: [thebatch.com](https://thebatch.com)

**TLDR AI**
- Frequency: Daily
- Time to read: 3-5 minutes
- Content: Brief AI news summaries
- Why: Stay current without commitment
- Subscribe: [tldr.tech/ai](https://tldr.tech/ai)

**Ahead of AI (Sebastian Raschka)**
- Frequency: Weekly-ish
- Time to read: 15-20 minutes
- Content: Deep technical analysis, research paper reviews
- Why: More technical than The Batch, excellent explanations
- Subscribe: [substack.com/ahead-of-ai](https://substack.com/ahead-of-ai)

---

### Great Podcasts

**Latent Space Podcast**
- Hosts: Swyx, Josh Pomer
- Frequency: Weekly, 60-90 minutes
- Content: Interview with AI engineers/founders
- Why: Practitioners teaching practitioners
- Where: All podcast apps, [latentspace.ai](https://latentspace.ai)

**The AI Podcast (NVIDIA)**
- Hosts: Research directors, founders
- Frequency: Weekly, 20-40 minutes
- Content: Technical deep-dives into recent advances
- Why: Short, concrete, high quality
- Where: All podcast apps

**No Priors Podcast**
- Hosts: Sarah Guo, Conviction (investors)
- Frequency: Weekly, 45 minutes
- Content: AI applications, startups, market analysis
- Why: Business perspective on technical advances
- Where: All podcast apps

---

## 5. Open Source Contribution Strategy

Contributing to open source:
- Accelerates your learning
- Builds your portfolio
- Turns you into community member (not just consumer)
- Sometimes leads to opportunities (maintainers remember you)

### How to Find Your First Issue

**Best projects to contribute to:**
1. **LlamaIndex** (RAG toolkit)
   - Why: Growing ecosystem, happy to help new contributors
   - Issues: Check "good first issue" label
   - Example: Fix documentation, add examples, improve error messages

2. **RAGAS** (RAG evaluation)
   - Why: Active development, critical to the ecosystem
   - Issues: Evaluation metrics, documentation
   - Example: "Add support for Hebrew language evaluation"

3. **sentence-transformers** (embedding models)
   - Why: Well-organized, clear docs
   - Issues: Training examples, new architectures
   - Example: "Add example for domain-specific fine-tuning"

4. **FastAPI** (web framework you use)
   - Why: Well-maintained, good starter issues
   - Issues: Documentation, examples, bug fixes
   - Example: "Update tutorial for streaming responses"

5. **Qdrant** (vector database)
   - Why: Growing, needs contributors
   - Issues: Client libraries, documentation
   - Example: "Add Python example for semantic caching"

6. **LangChain** (LLM framework)
   - Why: Huge ecosystem, lots of work
   - Issues: Integrations, documentation, examples
   - Example: "Add integration with new embedding model"

### Your First Contribution

**Step 1: Pick an issue**
- Find "good first issue" label
- Pick something 2-4 hours of work
- Comment: "I'd like to work on this"

**Step 2: Fork and clone**
```bash
git clone https://github.com/YOUR_FORK/project.git
cd project
git checkout -b fix/your-fix-name
```

**Step 3: Make the change**
```bash
# Edit files
# Test your change
pytest  # Run tests to ensure you didn't break anything
```

**Step 4: Commit and push**
```bash
git add .
git commit -m "fix: description of what you fixed"
git push origin fix/your-fix-name
```

**Step 5: Open a pull request**
- Describe what you fixed
- Reference the issue: "Closes #123"
- Be humble, ask for feedback

**Step 6: Respond to review**
- Maintainers might ask for changes
- Address them (this is learning)
- Push changes (no need to close PR, just update)

### Small vs Meaningful Contributions

**Typo fix:** Good first contribution, but doesn't build your skills
**Bug fix:** Good, you solve a real problem
**New feature:** Best, you understand the codebase deeply
**Documentation improvement:** Excellent, underrated

### How to Credit It

On your resume:
```
Open Source Contributor — [Project]
- Fixed bug in [component], improved [metric] by 10%
- Added support for [feature] used by 1K+ users
- Contributed to 5+ open-source projects
```

On your portfolio:
```
Link to your contributions: github.com/YOUR_NAME/contributions
```

In interviews:
```
"I've contributed to LlamaIndex and RAGAS. I was frustrated with 
documentation, fixed it, and now 50 other people use that doc."
```

---

## 6. From Junior to Senior: The 18-Month Accelerator

You're hired as a junior AI engineer. What makes you senior?

### Year 1 (Junior → Competent)

**What matters:**
- **Ship features, don't just read code**
  - Goal: Deliver 1 feature per sprint, end-to-end
  - Not: Understand 10 different parts, ship nothing
  
- **Own one system completely**
  - Goal: One API, one model, one database that you understand inside-out
  - Why: Ownership is how you learn
  - Not: Contribute to everything
  
- **Measure everything**
  - Set up your own dashboards for your systems
  - Track: latency, error rate, cost, accuracy
  - Why: Measurement reveals problems
  
- **Learn from incidents by reading postmortems**
  - Your company has outages/failures
  - Read the postmortem (not to blame, to learn)
  - Why: Failures are your best teacher
  
- **Ask "why?" about architectural decisions**
  - Why do we use this database?
  - Why is this the API design?
  - Why not use this new framework?
  - Why: Understanding reasoning teaches you design thinking

**Metrics of success:**
- [ ] Shipped 30+ features in your own area
- [ ] Own a system that handles >99% of requests you touch
- [ ] Know the codebase well enough to answer questions
- [ ] Can explain your system's performance characteristics

---

### Year 2 (Competent → Senior)

**What matters:**
- **Design your own system from scratch**
  - Goal: Take a new feature, design → implement → ship
  - Not: Copy existing architecture
  - Why: Design is the hardest skill
  
- **Mentor a junior engineer**
  - Goal: Help 1 junior person level up (1 hour/week)
  - Why: Teaching deepens your own understanding by 2x
  - Bonus: Hiring managers notice
  
- **Contribute to technical decisions**
  - In design meetings, you propose. Not just listen.
  - You push back on bad ideas with evidence
  - You advocate for your system's concerns
  
- **Build something new (not just maintain)**
  - Don't spend 2+ years fixing bugs in old code
  - Every 6 months, work on new feature/system
  - Why: Growth requires novelty
  
- **Present your work externally**
  - Write 1 blog post or give 1 talk
  - Topic: Something you solved that others face
  - Why: Clarity + communication are senior skills
  - Bonus: You're now known in the community

**Metrics of success:**
- [ ] Designed and shipped 1 new system end-to-end
- [ ] Mentored 1+ junior engineers
- [ ] Published 1 blog post or conference talk
- [ ] Can design a new system in an interview confidently

---

### What Makes Someone Senior (Beyond Year 2)

Senior isn't just "more years." It's:

**Knows when NOT to use AI** (saves company money)
- "We don't need an LLM for this; a simple rule works"
- "This isn't a ML problem; it's a data problem"
- Cost-awareness is rare and valuable

**Can debug a model degradation in production**
- Model performance dropped from 0.92 → 0.85 accuracy
- You know the 5 most likely causes
- You methodically test each one
- You find the issue and fix it in 2 hours, not 2 days

**Designs systems thinking about failure modes**
- "What happens when Redis is down?"
- "What if the LLM API returns garbage?"
- "How do we recover from a bad model deployment?"
- Junior: Didn't think about this
- Senior: Designed for it

**Evaluates vendor claims critically**
- Vendor says "our model is 2% better"
- You ask: "On what benchmark? How many runs? Different data?"
- You test before buying

**Can teach others — has strong mental models**
- You don't just know how to do something
- You understand WHY it works
- You can explain it to people with different backgrounds
- This is the mark of true mastery

---

## The Path Forward

This roadmap taught you the fundamentals.

But fundamentals are the beginning, not the end.

**Month 1 after finishing:** Pick one specialization, do 1-2 things from that path

**Month 2-3:** Deep dive into that path

**Month 6:** Decide if it's right for you, adjust

**Year 1:** Deepen your chosen specialization while staying broadly aware

**Year 2+:** Balance depth (in your specialty) with breadth (staying current)

---

## Communities, Newsletters, and Resources

| Resource | Type | Time | Frequency | Worth It? |
|----------|------|------|-----------|-----------|
| The Batch | Newsletter | 10 min | Weekly | ✅ Essential |
| TLDR AI | Newsletter | 3-5 min | Daily | ✅ Great for staying current |
| Latent Space | Podcast | 60-90 min | Weekly | ✅ Best conversations |
| HuggingFace Discord | Community | Variable | Always on | ✅ Essential |
| LangChain Discord | Community | Variable | Always on | ✅ Very helpful |
| MLOps Community | Community | Variable | Always on | ✅ Professional |
| Ahead of AI | Newsletter | 15-20 min | Weekly | ✅ Deep technical |
| AIEngineers Slack | Community | Variable | Always on | ✅ Practitioner network |

---

## The Honest Truth

You'll spend the next few years wondering if you're learning fast enough.

You won't be.

But that's fine.

**The people who succeed are the ones who:**
- Show up consistently
- Ship something every month
- Talk to other engineers
- Stay curious but not obsessive
- Remember why they started

You've built a strong foundation. Now you choose your path.

The AI industry needs:
- Researchers pushing capability
- Infrastructure engineers scaling reliability
- Product engineers making AI useful to humans

Pick what excites you. Get good at it. Help others.

---

## Your Next Week

- [ ] Join 1 community (Discord or Slack)
- [ ] Read 1 newsletter
- [ ] Listen to 1 podcast episode
- [ ] Identify your specialization interest
- [ ] Contribute to 1 open-source issue (even a typo fix)
- [ ] Write 1 blog post or Tweet about what you learned

Then start building depth.

---

**You're ready. The work starts now.**

**Go.**

---

← [Previous: Project Templates](project-template.md)

---

*This roadmap was designed by engineers, for engineers. Built through 20+ weeks of comprehensive learning. 39 files. 100,000+ words. 0 fluff.*

*You did the work. You earned this knowledge.*

*Now share it.*
