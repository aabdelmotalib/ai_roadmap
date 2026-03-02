# Phase 3 Milestone: MLOps Pipeline for AI Fine-Tuning

You've learned to deploy systems (Phase 2) and manage iterations (Phase 3).

This milestone validates you can **systematically improve AI systems** in production.

---

## Exit Criteria Checklist (20+ Items)

### MLflow (Experiment Tracking)

- [ ] Can log parameters, metrics, artifacts with MLflow
- [ ] Have run 10+ experiments and compared them
- [ ] Registered best model in Model Registry
- [ ] Know difference between runs, params, and metrics
- [ ] Can retrieve and evaluate historical runs
- [ ] Have used MLflow Server (local or remote)
- [ ] Understand artifact storage (S3 backend)

### DVC (Data Versioning)

- [ ] Initialized DVC in a git repo
- [ ] Added large file to DVC, pushed to remote
- [ ] Created dvc.yaml pipeline with 3+ stages
- [ ] Reproduced an experiment from git history
- [ ] Understand dvc.lock and reproducibility
- [ ] Have versioned both code, data, and model
- [ ] Know how to debug DVC issues (cache, remote, etc)

### Weights & Biases (Visualization)

- [ ] Logged training runs with W&B
- [ ] Created interactive dashboard
- [ ] Ran hyperparameter sweep (3+ combinations)
- [ ] Set up alerts for metric drops
- [ ] Understand W&B tables for sample evaluation
- [ ] Can export results for reports

### GitHub Actions (CI/CD)

- [ ] Created workflow that triggers on push
- [ ] Workflow runs training automatically
- [ ] Have stored secrets (API keys) safely
- [ ] Understand conditional deployment (only if metric improves)
- [ ] Can debug workflow failures
- [ ] Have received Slack notification from workflow

### Production MLOps

- [ ] Pipeline is fully automated (no manual retraining)
- [ ] Metrics tracked across all runs
- [ ] Can compare any two models systematically
- [ ] Experiment cost tracked and under control
- [ ] Baseline established and monitored
- [ ] Team can reproduce any experiment
- [ ] Rollback to previous model is < 5 minutes

---

## Capstone Project: "MLOps Pipeline for AI Fine-Tuning"

**What You'll Build:** End-to-end pipeline that:
1. Auto-trains on new data
2. Auto-evaluates (RAGAS)
3. Compares to baseline
4. Auto-deploys if metric improves
5. Tracks everything

### Architecture

```
GitHub: Code + DVC pointers
  ↓
GitHub Actions: Tests + trains
  ↓
MLflow: Logs parameters, metrics
  ↓
DVC: Versions data, models
  ↓
W&B: Rich visualization
  ↓
Auto-deploy: If metric improves
  ↓
Production: New model serving
```

### Requirements

**MLflow:**
- [ ] Tracks 10+ experiments
- [ ] Compares by RAGAS faithfulness
- [ ] Model Registry with 3+ versions
- [ ] Can load best model for inference

**DVC:**
- [ ] dvc.yaml with pipeline: prepare → train → evaluate
- [ ] Versioned training data (train.jsonl)
- [ ] Versioned models (model.safetensors)
- [ ] Reproducible: `git checkout` + `dvc pull` + `dvc repro` = same results

**Weights & Biases:**
- [ ] Training logged with loss curves
- [ ] Evaluation logged with RAGAS metrics
- [ ] Table showing sample predictions
- [ ] Dashboard comparing runs
- [ ] Alert if faithfulness < 0.75

**GitHub Actions:**
- [ ] Workflow on push/schedule
- [ ] Trains with `python train.py`
- [ ] Evaluates with RAGAS
- [ ] Compares to baseline
- [ ] If improved: deploys to production
- [ ] Slack notification on completion
- [ ] Cost tracking (logs training time)

**Production:**
- [ ] Model served in ECS (from Phase 2)
- [ ] Can rollback in < 5 minutes
- [ ] Monitoring shows model age
- [ ] Alert if baseline model serving (rollback happened)

### Deliverables

```
├── train.py                      # Training script (logged to MLflow)
├── evaluate.py                   # Evaluation (logs to W&B)
├── dvc.yaml                      # Pipeline definition
├── params.yaml                   # Hyperparameters
├── .github/workflows/
│   └── train-and-deploy.yml      # GitHub Actions workflow
├── baseline_metrics.json         # Baseline for comparison
├── requirements.txt
├── deployment-guide.md           # How to use the pipeline
└── README.md                     # Architecture, how everything connects
```

---

## Self-Assessment Rubric

### Beginner (Not Ready Yet)

**Indicators:**
- Can follow tutorial, but struggle to modify
- Pipeline works once, hard to repeat
- Don't understand how MLflow/DVC connect
- Metrics buried, hard to find
- Unlikely to share with team

**Example:** "I got MLflow working, but I don't know how to integrate with GitHub Actions."

### Competent (Ready for Phases 4+)

**Indicators:**
- Can write training script with MLflow integration
- DVC pipeline is reproducible
- GitHub Actions workflow auto-deploys
- Can compare models systematically
- Baseline established, monitoring in place
- Team can understand your pipeline

**Example:** "I have DVC tracking data, MLflow tracking experiments, W&B visualization, and GitHub Actions that auto-deploys if RAGAS > 0.80."

### Hire-Ready (Exceptional)

**Indicators:**
- Full MLOps lifecycle automated
- Cost optimized (efficient experiments)
- Metrics SLOs defined (what's acceptable?)
- Data drift detected (alert if quality drops)
- A/B testing infrastructure ready
- Can onboard new teammate to pipeline in 1 hour
- Systematic improvement over time (not random)

**Example:** "Pipeline tracks data drift, auto-retrains if quality drops. We have 200 experiments over 6 months, consistently improving baseline. Cost $150/month. Team can reproduce any experiment in 1 hour."

---

## What a Failed Milestone Looks Like

⚠️ **You're not ready if:**

1. ❌ "I have MLflow but don't use it"
   - If you're not logging experiments, you don't have MLOps. You have training code.

2. ❌ "DVC is too complex, I just copy files"
   - Then you can't reproduce. You need versioning.

3. ❌ "Experiments are random, I don't know what changed"
   - Every run must log: data version, code version, hyperparameters, metrics, timestamp.

4. ❌ "No baseline, can't know if improvement is real"
   - Metric could just be variance. Need baseline to compare.

5. ❌ "GitHub Actions times out, I gave up"
   - Scale compute (longer timeout), cache dependencies, split into stages.

6. ❌ "Manual deployment, waiting for ops to push to prod"
   - You should auto-deploy. That's the point of CI/CD.

7. ❌ "Team doesn't know how the pipeline works"
   - Document it. Write a 1-page guide. Make it repeatable by others.

8. ❌ "Costs $500/month in compute"
   - Experiments are inefficient. Use smaller models, shorter iterations, pre-validation.

---

## Common Reasons People Rush Phase 3

### "MLflow is just logging, I'll skip it"
**Consequence:** Months later: "Which model is in production?" Nobody knows.
**Reality:** MLflow saves you for 1-2 hours of setup. Pays back constantly after.

### "DVC is complex, Git is enough"
**Consequence:** Data grows to 100GB. Git repo is unusable.
**Reality:** DVC takes 2 days to learn. You'll use it for years.

### "Sweeps are too slow, I'll just try one config"
**Consequence:** Pick worst hyperparameters. Performance is mediocre.
**Reality:** Sweeps run in parallel. Sample 12 configs in 6 hours, find optimal.

### "GitHub Actions is overkill, I'll train manually"
**Consequence:** Manual training = human error. Forgot to update metrics. Deployed wrong model.
**Reality:** Automation removes human error. Takes 1 day to set up.

### "I don't need monitoring, the model is good"
**Consequence:** Model performance drifts. You don't notice. Real-world quality drops.
**Reality:** Set baselines. Monitor. Alert. Know immediately if something's wrong.

---

## What's in Phase 4 (Kubernetes & Advanced Ops)

Phase 4 (4-6 weeks): **Kubernetes & Multi-Region Deployment**

By now:
- You deploy AI systems (Phase 2)
- You iterate systematically (Phase 3)

In Phase 4, you'll **operate at massive scale**:
- Kubernetes: Orchestrate 100s of containers
- Multi-region: Serve globally (latency + resilience)
- Advanced monitoring: Distributed tracing, performance budgets
- Cost optimization: Resource quotas, autoscaling policies
- Disaster recovery: Backup, failover, recovery RTO/RPO

Example: Netflix runs 1000s of models in production. You'll learn how.

→ **[Phase 4: Kubernetes & Advanced Ops →](../../phase4/overview.md)**

---

## Final Thoughts

Now you have the full picture:

**Phase 1:** How to build AI systems (APIs, embeddings, RAG, agents, evaluation)
**Phase 2:** How to deploy them (Docker, AWS, monitoring)
**Phase 3:** How to improve them (MLOps, versioning, automation)

Three phases. 12+ weeks. You've learned enough to:
- Build production AI systems
- Deploy them reliably
- Improve them systematically
- Collaborate with teams
- Operate at scale

You're an **AI Engineer** now.

---

## Deployment Checklist (Before Going Live)

- [ ] MLflow tracks all experiments (baseline + 5+ runs)
- [ ] DVC versions data and models (reproducible)
- [ ] GitHub Actions auto-trains on schedule
- [ ] Evaluation metrics in W&B (visible to team)
- [ ] Baseline established and documented
- [ ] Alert thresholds set (email + Slack)
- [ ] Cost per experiment < $10
- [ ] Rollback is < 5 minutes
- [ ] Team documentation complete
- [ ] New teammate can run full pipeline in 1 hour

---

## 1-Week Phase 3 Review

- [ ] Day 1: MLflow setup and log first experiment
- [ ] Day 2: DVC pipeline setup
- [ ] Day 3: W&B dashboard creation
- [ ] Day 4: GitHub Actions workflow
- [ ] Day 5: Auto-deploy integration
- [ ] Day 6: Team review and feedback
- [ ] Day 7: Document and test reproducibility

---

## Resources

- [MLflow Guide](https://mlflow.org/docs/)
- [DVC Handbook](https://dvc.org/learn)
- [W&B Tutorials](https://docs.wandb.ai/tutorials)
- [GitHub Actions Best Practices](https://docs.github.com/en/actions)

---

**Congratulations on Phases 1-3.** 🎓

You've completed **20+ weeks of AI systems engineering**.

You understand:
- How to build AI systems
- How to deploy them
- How to improve them systematically

Next: Scaling to production (Phase 4).

But first: **Build something**. Apply what you've learned. Ship to production. Get users. Iterate.

The learning never stops. But you now have the foundation.

**Go ship.** 🚀
