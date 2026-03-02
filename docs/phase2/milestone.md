# Phase 2 Milestone: Containerized AI Service on AWS

You've learned to containerize and deploy AI systems to production-grade infrastructure.

This milestone validates you can go from code on your laptop to running in AWS.

---

## Exit Criteria Checklist (20+ Items)

### Docker & Containerization

- [ ] Can write a production Dockerfile from scratch
- [ ] Understand multi-stage builds and why they matter
- [ ] Know the difference between COPY, CMD, ENTRYPOINT
- [ ] Can optimize layer caching (requirements first, code last)
- [ ] Know which base image to use for different scenarios
- [ ] Understand .dockerignore and what to exclude
- [ ] Can run `docker compose up` and have full stack work
- [ ] Have written and tested a Dockerfile locally

### Docker Compose

- [ ] Can write docker-compose.yml with 4+ services
- [ ] Understand volumes, networks, environment variables
- [ ] Know difference between named volumes and bind mounts
- [ ] Can use healthchecks to ensure service readiness
- [ ] Understand `depends_on` and race condition prevention
- [ ] Have tested full local stack (API + DB + Cache + Vector DB)

### AWS Deployment

- [ ] Have created AWS account and IAM user
- [ ] Can push Docker image to ECR
- [ ] Understand EC2, ECS Fargate, RDS concepts
- [ ] Have deployed to ECS (or EC2)
- [ ] Know how to configure ALB and security groups
- [ ] Understand auto-scaling triggers
- [ ] Have stored files in S3 successfully

### Secrets & Monitoring

- [ ] Know to never hardcode API keys
- [ ] Have used AWS Secrets Manager
- [ ] Know difference between Secrets Manager and Parameter Store
- [ ] Can implement structured logging (JSON format)
- [ ] Have set up CloudWatch alarms
- [ ] Understand what metrics to monitor (error rate, latency, cost)
- [ ] Can query CloudWatch Logs Insights

### Production Readiness

- [ ] Have tested service locally before AWS
- [ ] Know how to troubleshoot Docker and AWS errors
- [ ] Understand health checks and why they're critical
- [ ] Can estimate monthly costs
- [ ] Know how to scale systems (horizontal vs vertical)

---

## Capstone Project: "Containerized AI Service on AWS"

**What You'll Build:** Deploy Phase 1's AI Document Intelligence Platform to production on AWS.

### Requirements

**Infrastructure:**
- Docker image of Phase 1 service (multi-stage, optimized)
- Docker Compose for local testing
- ECS Fargate deployment (3+ tasks for high availability)
- RDS PostgreSQL (Multi-AZ for durability)
- ElastiCache Redis cluster
- Qdrant vector DB (or pgvector in RDS)
- Application Load Balancer (auto-scaling with CPU metric)
- S3 bucket for model artifacts and documents
- Secrets Manager for API keys and database credentials
- CloudWatch monitoring with 3+ alarms (error rate, latency, cost)

**API Endpoints (same as Phase 1, but now in production):**
```
POST   /documents/upload      # Upload documents to S3
POST   /documents/{id}/delete
GET    /search                # Semantic search
POST   /query                 # Generate answer with monitoring
POST   /feedback              # Log user ratings
GET    /health                # Health check for ALB
GET    /costs/daily           # Cost breakdown
GET    /status                # Service status
```

**Success Criteria:**
- [ ] All services deploy successfully via CloudFormation or CLI
- [ ] Service handles 10+ concurrent requests without crashing
- [ ] RAGAS score maintained at > 0.75 on test dataset
- [ ] Response time: p99 < 5 seconds
- [ ] Cost: < $200/month for expected traffic
- [ ] Zero hardcoded secrets in code or images
- [ ] CloudWatch logs show all requests properly structured
- [ ] Auto-scaling works (verified by load test)
- [ ] Can rollback to previous version in < 5 minutes
- [ ] Have runbook for common issues (OOM, DB connection pool exhausted)

### Implementation Steps

1. **Containerize Phase 1 Service**
   - Write Dockerfile (multi-stage)
   - Test locally with `docker run`
   - Create docker-compose-prod.yml

2. **Set Up AWS Infrastructure**
   - Create RDS PostgreSQL instance
   - Create ElastiCache Redis cluster
   - Create S3 buckets
   - Create IAM roles and policies

3. **Deploy to ECS Fargate**
   - Create ECR repository
   - Push image to ECR
   - Create ECS cluster and task definition
   - Create ECS service with ALB
   - Configure auto-scaling

4. **Monitoring & Alerts**
   - Set up CloudWatch log group
   - Implement structured logging
   - Create CloudWatch alarms
   - Verify alerts work (send test)
   - Create Slack integration

5. **Testing**
   - Load test with 100 concurrent users
   - Verify auto-scaling triggers
   - Test failover (kill a task, verify recovery)
   - Test full deployment process (ability to repeat)

### Deliverables

```
├── Dockerfile                # Production Dockerfile
├── docker-compose.yml         # Local dev
├── docker-compose.prod.yml    # AWS-ready
├── .dockerignore              # What to exclude
├── task-definition.json       # ECS configuration
├── iam-policy.json           # Least-privilege IAM
├── deployment-guide.md        # How to deploy (step by step)
├── architecture.md            # System diagram
├── monitoring-dashboard.json  # CloudWatch dashboard
└── runbook.md                # How to handle common failures
```

---

## Self-Assessment Rubric

### Beginner (Not Ready for Phase 3)

**Indicators:**
- Can follow Docker tutorial, but struggles to modify Dockerfile
- Deployment works on AWS, but don't understand why
- Haven't integrated with secrets or monitoring
- Copy-paste solution without comprehending it
- Can't troubleshoot when deployment fails

**Example:** "I deployed to ECS, but it keeps failing health checks. I don't know why."

### Competent (Ready for Phase 3)

**Indicators:**
- Can write Dockerfile without reference
- Understand image optimization (layers, caching, multi-stage)
- Have successfully deployed to AWS and it works
- Implemented proper secrets management
- Monitoring is set up with meaningful alerts
- Can debug Docker and AWS errors
- Estimated and understand monthly costs

**Example:** "I deployed Phase 1 service with Dockerfile, RDS, Redis, ECS Fargate, ALB, and CloudWatch monitoring. It scales automatically and costs $150/month."

### Hire-Ready (Exceptional)

**Indicators:**
- Containerization is optimized (image < 300MB, builds in < 3 min)
- All infrastructure defined in code (Terraform or CloudFormation)
- Security best practices: secrets never in code, proper IAM, no root user
- Comprehensive monitoring: error budgets, cost alerts, performance SLOs
- Can explain every AWS decision and alternatives considered
- Handled edge cases (OOM mitigation, connection pool sizing)
- Crisis recovery: can revert failed deployment in < 5 minutes

**Example:** "I containerized with BuildKit for parallel builds, deployed with Terraform, configured auto-scaling based on p99 latency, secured with IAM roles per service, and set up cost alerts at 80% of $200 budget."

---

## What a Failed Milestone Looks Like

⚠️ **You're not ready for Phase 3 if:**

1. ❌ "My Dockerfile works but it's 2GB"
   - Multi-stage builds dramatically reduce size. You haven't optimized.

2. ❌ "I don't know why the health check keeps failing"
   - Health checks are critical. If you can't debug this, you'll struggle in production.

3. ❌ "Deployment works, but I don't understand why"
   - You need to know each piece: Docker → ECR → ECS → ALB → RDS.

4. ❌ "I hardcoded the database password in the Dockerfile"
   - Security risk. Secrets Manager exists for a reason.

5. ❌ "No idea what's in CloudWatch logs"
   - Logging is essential. If you can't read logs, you can't debug production issues.

6. ❌ "My service works locally but fails on AWS after 100 requests"
   - Likely connection pool exhaustion. You need to understand resource limits.

7. ❌ "Running on 1 instance, no auto-scaling"
   - AWS gives you auto-scaling for free. If you'm not using it, you're not using AWS correctly.

8. ❌ "AWS bill is $500/month for a side project"
   - You don't understand cost optimization. Should be < $100.

---

## Common Reasons People Rush Phase 2 (And Consequences)

### "Docker is just containers, I'll figure it out later"
**Consequence:** Dockerfile bloats to 5GB. Deploys take 30 minutes. Team gets frustrated.
**Reality:** Spend 2 days now optimizing, save 1 hour per week forever.

### "Monitoring is optional, I'll add it after launch"
**Consequence:** Service crashes in production. You don't know until customer complains.
**Reality:** Monitoring should be installed first. You can monitor broken code too.

### "I'll use root user in Docker, security is for later"
**Consequence:** Attacker gains access to your instance with root privileges.
**Reality:** Non-root user is 10 lines in Dockerfile. Do it now.

### "Skip Secrets Manager, just use environment variables"
**Consequence:** API key checked into git. Attacker finds it. Your account gets hacked.
**Reality:** Secrets Manager takes 30 min. Your security depends on it.

### "Copy StackOverflow ECS configuration, don't understand IAM"
**Consequence:** Service has admin access to all AWS resources. Breach exposes everything.
**Reality:** Least-privilege IAM is a principle. Learn it properly.

### "Auto-scaling is complex, I'll just provision 1 large instance"
**Consequence:** Service dies if traffic spikes. Or costs $500/month.
**Reality:** Auto-scaling is 5-line policy. Use it.

---

## What's in Phase 3

Phase 3 (2-3 weeks): **MLOps Pipelines & Experiment Tracking**

In Phase 2, you deployed systems. In Phase 3, you'll **manage iterations** of those systems.

You'll learn:
- **MLflow**: Track every fine-tuning experiment
- **DVC**: Version your datasets and models
- **Weights & Biases**: Visualize training and compare runs
- **GitHub Actions**: CI/CD that auto-deploys best model
- **Cost tracking**: Know what each experiment costs

Example workflow:
1. Data scientist runs fine-tuning experiment (logged to MLflow)
2. Evaluation script auto-runs (RAGAS metrics)
3. If metric improves over baseline: auto-deploy to staging
4. If metrics hold after 1 week: promote to production
5. All tracked in W&B dashboard with cost attribution

→ **[Phase 3: MLOps & Experiment Tracking →](../../phase3/overview.md)**

---

## Final Thoughts

Phase 2 was about **reliability and scale**.

In Phase 1, you built an AI system.
In Phase 2, you built a **production AI system**.

The difference:
- Phase 1: Works on your laptop
- Phase 2: Works on others' laptops, handles scale, survives software updates, has observability

You're now an **AI engineer who can ship**.

Next: You'll become one who can **iterate** systematically (Phase 3).

---

## Deployment Checklist (Before Launching)

- [ ] Dockerfile gets to < 500MB (preferably < 300MB)
- [ ] No secrets in Dockerfile or code
- [ ] Health check working (curl /health returns 200)
- [ ] Services in ECS have proper resource limits
- [ ] RDS backups enabled (daily)
- [ ] CloudWatch alarms set (error rate, latency, cost)
- [ ] Load balancing configured with > 2 tasks
- [ ] Auto-scaling policy in place
- [ ] SSL/TLS certificates set up (HTTPS)
- [ ] Runbook documented (how to fix common failures)
- [ ] Tested rollback (can revert version in < 5 min)
- [ ] Cost estimated and within budget
- [ ] Monitoring dashboards accessible
- [ ] On-call schedule in place (who gets paged at 3am?)

---

## Resources

- [AWS Best Practices](https://docs.aws.amazon.com/best-practices/)
- [ECS Task Definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)
- [Docker Optimization](https://docs.docker.com/build/optimize/)

---

**Congratulations on Phase 2.** 🎓

You've deployed production systems. You understand infrastructure. You can operate at scale.

Next stop: Automating your AI iterations.
