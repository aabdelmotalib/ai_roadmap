# CI/CD for AI Pipelines with GitHub Actions

CI/CD: Continuous Integration / Continuous Deployment.

Normal: Push code → Manual testing → Someone deploys → Fingers crossed

CI/CD: Push code → Auto-run tests → If good → Auto-deploy → No fingers needed

For AI: Push code → Auto-train → Auto-evaluate → If metric improves → Auto-deploy

---

## Why This Matters

**Real incident:** Data scientist fine-tunes model. Pushes to repo. Forgets to update production. Waits 2 weeks to ask ops to deploy.

CI/CD: Pushes to repo → Automatically fine-tunes → Automatically evaluates → Automatically deploys if better.

Instead of 2 weeks: 2 hours.

---

## Conceptual Explanation

**Analogy: QA Testing on Assembly Line**

Without CI/CD: Build car → Ship to customer → Hope it works

With CI/CD: Build car → Auto-test brakes → Auto-test steering → Auto-test lights → Only ship if all pass

---

## GitHub Actions Basics

GitHub Actions: Workflows triggered by events (push, PR, schedule).

```yaml
name: Train and Deploy

on:
  push:
    branches:
      - main

jobs:
  train:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install deps
        run: pip install -r requirements.txt
      
      - name: Pull data
        run: dvc pull
      
      - name: Train
        run: python train.py
      
      - name: Evaluate
        run: python evaluate.py
      
      - name: Deploy
        if: success()
        run: |
          aws ecs update-service --cluster ai-cluster \
            --service ai-service \
            --force-new-deployment
```

---

## Code Example 1: Full ML Pipeline Workflow

```yaml
name: ML Pipeline

on:
  push:
    branches: [main, develop]
  schedule:
    - cron: '0 0 * * 0'  # Weekly

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com
  ECR_REPOSITORY: ai-service

jobs:
  train-and-evaluate:
    runs-on: ubuntu-latest
    timeout-minutes: 120
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: us-east-1
      
      - name: Cache pip dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Pull data with DVC
        run: dvc pull -R
      
      - name: Run training
        run: python src/train.py
        env:
          WANDB_API_KEY: ${{ secrets.WANDB_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Evaluate model
        run: |
          python src/evaluate.py
          cat metrics.json
      
      - name: Check metric threshold
        run: |
          FAITHFULNESS=$(python -c "import json; m=json.load(open('metrics.json')); print(m['ragas_faithfulness'])")
          if (( $(echo "$FAITHFULNESS < 0.75" | bc -l) )); then
            echo "Metric below threshold: $FAITHFULNESS"
            exit 1
          fi
          echo "Metric OK: $FAITHFULNESS"
      
      - name: Upload metrics to W&B
        run: python -c "import wandb; wandb.init(project='nlp'); wandb.log(json.load(open('metrics.json')))"
        if: always()
      
      - name: Build Docker image
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${{ github.sha }} .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:${{ github.sha }} \
            $ECR_REGISTRY/$ECR_REPOSITORY:latest
      
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:${{ github.sha }}
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster ai-cluster \
            --service ai-service \
            --force-new-deployment \
            --region us-east-1
      
      - name: Slack notification
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Training job completed'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Code Example 2: Pull Request Quality Gate

```yaml
name: PR Quality Check

on:
  pull_request:
    branches: [main]

jobs:
  evaluate:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install deps
        run: pip install -r requirements.txt
      
      - name: Get baseline metrics
        run: |
          git fetch origin main
          git checkout main
          dvc pull
          python evaluate.py
          mv metrics.json metrics_baseline.json
          cat metrics_baseline.json
      
      - name: Checkout PR changes
        run: |
          git checkout -
          dvc pull
      
      - name: Evaluate new version
        run: python evaluate.py
      
      - name: Compare metrics
        run: |
          python -c "
          import json
          
          with open('metrics_baseline.json') as f:
            baseline = json.load(f)
          
          with open('metrics.json') as f:
            current = json.load(f)
          
          # Check if worse than baseline
          delta = current['ragas_faithfulness'] - baseline['ragas_faithfulness']
          
          print(f'Baseline: {baseline[\"ragas_faithfulness\"]:.3f}')
          print(f'Current:  {current[\"ragas_faithfulness\"]:.3f}')
          print(f'Delta:    {delta:+.3f}')
          
          if delta < -0.05:  # Allow 5% drop
            print('❌ Quality degradation detected!')
            exit(1)
          else:
            print('✅ Quality acceptable')
          "
      
      - name: Comment PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const metrics = JSON.parse(fs.readFileSync('metrics.json', 'utf8'));
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Model Evaluation Results\n\n- Faithfulness: ${metrics.ragas_faithfulness.toFixed(3)}\n- Relevancy: ${metrics.ragas_relevancy.toFixed(3)}\n- Context Recall: ${metrics.context_recall.toFixed(3)}`
            })
```

---

## Code Example 3: Scheduled Training (Weekly)

```yaml
name: Weekly Training

on:
  schedule:
    # Every Sunday at 2 AM UTC
    - cron: '0 2 * * 0'

jobs:
  weekly-retrain:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install deps
        run: pip install -r requirements.txt
      
      - name: Pull latest data
        run: dvc pull
      
      - name: Train with fresh data
        run: python train.py
        env:
          EXPERIMENT_NAME: "weekly-retraining"
          WANDB_API_KEY: ${{ secrets.WANDB_API_KEY }}
      
      - name: Auto-deploy if improved
        run: |
          python -c "
          import json
          import subprocess
          
          # Load metrics
          with open('metrics.json') as f:
            metrics = json.load(f)
          
          # Load baseline
          with open('baseline_metrics.json') as f:
            baseline = json.load(f)
          
          if metrics['ragas_faithfulness'] > baseline['ragas_faithfulness']:
            print('✅ Improvement detected, deploying...')
            subprocess.run(['aws', 'ecs', 'update-service', ...])
          else:
            print('❌ No improvement, keeping current model')
          "
```

---

## Code Example 4: Secrets in GitHub Actions

```yaml
jobs:
  train:
    runs-on: ubuntu-latest
    steps:
      - name: Train with secrets
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          export OPENAI_API_KEY
          export DATABASE_URL
          python train.py
```

**Store secrets:**
- Go to: Settings → Secrets and variables → Actions
- Add: OPENAI_API_KEY, DATABASE_URL, WANDB_API_KEY, AWS_ROLE_ARN

---

## Practical Project: Auto-Deploying ML Pipeline

**Build:** GitHub Actions workflow that:
1. Triggers on push to main
2. Trains model with new data
3. Evaluates (RAGAS)
4. Compares to baseline
5. If improved: auto-deploys to ECS
6. If degraded: creates alert
7. Logs to W&B

---

## Debugging: Common Workflow Issues

### 1. Workflow Timeout

```yaml
jobs:
  train:
    timeout-minutes: 180  # Increase if training is slow
```

### 2. Secret Not Found

```yaml
# Secret name case-sensitive
env:
  OPENAI_KEY: ${{ secrets.openai_key }}  # Wrong: lowercase

env:
  OPENAI_KEY: ${{ secrets.OPENAI_API_KEY }}  # Right
```

### 3. Out of Disk Space

```yaml
- name: Free disk
  run: |
    sudo apt-get clean
    docker rmi -a
```

### 4. Insufficient Memory for Training

```yaml
jobs:
  train:
    runs-on: ubuntu-latest-4-cores  # Upgrade runner
```

### 5 DVC Pull Hangs

```yaml
- name: DVC pull with timeout
  run: timeout 600 dvc pull || true  # Don't fail if timeout
```

---

## Case Study: HuggingFace Transformer Model Deployment

**Challenge:** New model version daily. How to automatically test and deploy?

**Solution:**
- GitHub Actions on each commit
- Auto-runs GLUE evaluation
- If score improves: update model card + deploy
- If score degrades: creates issue, alerts team

**Result:** Continuous improvement pipeline, 1-day automated cycle

---

## 1-Week GitHub Actions Checklist

- [ ] Day 1: Create first workflow (echo "Hello")
- [ ] Day 2: Add DVC pull + training step
- [ ] Day 3: Add evaluation and metrics comparison
- [ ] Day 4: Set up secrets (OPENAI_API_KEY, etc)
- [ ] Day 5: Add Docker build and push step
- [ ] Day 6: Deploy to ECS on success
- [ ] Day 7: Add Slack notifications

---

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions/)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [AWS Credentials Action](https://github.com/aws-actions/configure-aws-credentials)

→ **[Phase 3 Milestone →](milestone.md)**
