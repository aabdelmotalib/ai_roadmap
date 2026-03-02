# MLflow: Experiment Tracking for AI

Every time you fine-tune or train an AI model, MLflow automatically logs:
- Parameters (learning rate, batch size, number of epochs)
- Metrics (RAGAS scores, loss, accuracy)
- Artifacts (model files, evaluation reports, plots)

Instead of trying to remember if run 47 was better than run 51, MLflow keeps perfect records.

---

## Why This Matters

**Without MLflow:** 50 fine-tuning experiments. You name them:
- model_v1.pt
- model_final.pt
- model_final_REAL.pt
- model_for_prod_maybe.pt
- model_actually_best.pt

Nobody knows which is actually best. It's 3am. Pager goes off. You have 30 seconds to figure out which model to rollback to.

**With MLflow:** Every run has a UUID, parameters, metrics, timestamp. You can compare instantly.

---

## Conceptual Explanation

**Analogy: Lab Notebook for Computer Science**

A chemist runs 100 experiments. Each one is documented:
- Date and time
- Exact chemicals used (parameters)
- Temperature, pressure (conditions)
- Results measured (metrics)
- Samples preserved for later (artifacts)

Years later, they can replicate any experiment.

MLflow is your lab notebook for AI experiments.

---

## Core MLflow Concepts

| Concept | What | | 
|---------|------|---|
| **Experiment** | Logical grouping (e.g., "LoRA fine-tuning") | Like a project |
| **Run** | Single training execution | Like a single lab experiment |
| **Parameter** | Fixed configuration (learning_rate, batch_size) | Input |
| **Metric** | Measured value (loss, RAGAS score) | Output |
| **Artifact** | Saved file (model, plot, report) | Physical sample |
| **Tag** | Human label (stage: "production") | Metadata |
| **Model Registry** | Central storage for models | Like model versioning |

---

## Code Example 1: Basic MLflow Tracking

```python
import mlflow
import mlflow.pytorch
from transformers import TrainingArguments, Trainer
import torch

# Set experiment
mlflow.set_experiment("lora-fine-tuning")

# Start a run
with mlflow.start_run(run_name="lr-1e4-r8"):
    
    # Log parameters
    mlflow.log_param("learning_rate", 1e-4)
    mlflow.log_param("lora_r", 8)
    mlflow.log_param("lora_alpha", 16)
    mlflow.log_param("batch_size", 32)
    mlflow.log_param("epochs", 3)
    
    # Training
    training_args = TrainingArguments(
        output_dir="./checkpoints",
        learning_rate=1e-4,
        num_train_epochs=3,
        per_device_train_batch_size=32,
        logging_steps=10
    )
    
    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=eval_dataset
    )
    
    # Train
    trainer.train()
    
    # Log metrics after training
    mlflow.log_metric("final_loss", trainer.state.best_loss)
    mlflow.log_metric("final_eval_loss", trainer.state.best_metric)
    
    # Log model
    mlflow.pytorch.log_model(model, "model")
    
    # Log evaluation report
    import json
    eval_report = {
        "accuracy": 0.95,
        "ragas_faithfulness": 0.88
    }
    mlflow.log_dict(eval_report, "evaluation.json")
    
    # Log artifact (file)
    mlflow.log_artifact("plots/loss_curve.png")

# Run ends automatically
print("Logged to MLflow!")
```

---

## Code Example 2: MLflow with HuggingFace Trainer

```python
from transformers import Trainer, TrainingArguments
from mlflow.transformers import autolog
import mlflow

# Auto-log everything from Trainer
mlflow.transformers.autolog()

mlflow.set_experiment("huggingface-tuning")

with mlflow.start_run():
    
    training_args = TrainingArguments(
        output_dir="./results",
        num_train_epochs=3,
        per_device_train_batch_size=8,
        learning_rate=5e-5,
        logging_steps=100,
        eval_steps=500,
        save_steps=1000
    )
    
    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=eval_dataset,
        compute_metrics=compute_metrics  # Will auto-log
    )
    
    trainer.train()
    
    # All metrics auto-logged!
    # No manual mlflow.log_metric() needed
```

---

## Code Example 3: Comparing Runs

```python
import mlflow
from mlflow import MlflowClient

client = MlflowClient()

# Get all runs in experiment
experiment = mlflow.get_experiment_by_name("lora-fine-tuning")
runs = client.search_runs(experiment_id=experiment.experiment_id)

# Find best run by accuracy
best_run = max(runs, key=lambda r: r.data.metrics.get("accuracy", 0))

print(f"Best run: {best_run.info.run_id}")
print(f"Params: {best_run.data.params}")
print(f"Metrics: {best_run.data.metrics}")

# Get all runs, sorted by metric
runs_sorted = sorted(
    runs,
    key=lambda r: r.data.metrics.get("ragas_faithfulness", 0),
    reverse=True
)

for i, run in enumerate(runs_sorted[:3]):
    print(f"#{i+1}: {run.data.metrics['ragas_faithfulness']:.3f}")
```

---

## Code Example 4: Model Registry

```python
import mlflow

# Register model from run
run_id = "abc123"
model_uri = f"runs:/{run_id}/model"

mlflow.register_model(
    model_uri=model_uri,
    name="lora-adapter-v2"
)

# Later: load registered model
model = mlflow.pytorch.load_model(f"models:/lora-adapter-v2/production")

# Or latest version
model = mlflow.pytorch.load_model(f"models:/lora-adapter-v2/latest")

# Transition between stages
client = mlflow.tracking.MlflowClient()

client.transition_model_version_stage(
    name="lora-adapter-v2",
    version=2,
    stage="staging"
)

# Later: promote to production
client.transition_model_version_stage(
    name="lora-adapter-v2",
    version=2,
    stage="production"
)
```

---

## Code Example 5: MLflow Server

```bash
# Start local MLflow server
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlruns \
  --host 0.0.0.0 \
  --port 5000

# Now visit http://localhost:5000
```

Production setup:
```bash
# Backend: RDS
# Artifacts: S3

mlflow server \
  --backend-store-uri postgresql://user:pass@rds.amazonaws.com/mlflow \
  --default-artifact-root s3://my-bucket/mlflow \
  --host 0.0.0.0 \
  --port 5000
```

---

## MLflow vs Alternatives

| Tool | Simplicity | Visualization | Team Features | Self-Hosted | Best For |
|------|-----------|---|---|---|---|
| **MLflow** | Easy | Basic | Limited | Yes | Simple tracking, local |
| **Weights & Biases** | Medium | Excellent | Great | No | Team collaboration, rich viz |
| **Neptun.ai** | Medium | Good | Good | No | Enterprise |
| **Kubeflow** | Hard | Good | Complex | Yes | Kubernetes-heavy |

---

## Step-by-Step: Set Up MLflow Locally

### 1. Install

```bash
pip install mlflow mlflow-transformers
```

### 2. Create Experiment

```python
import mlflow

mlflow.set_experiment("my-first-experiment")
```

### 3. Log a Run

```python
with mlflow.start_run(run_name="baseline"):
    mlflow.log_param("model", "gpt-4o-mini")
    mlflow.log_metric("accuracy", 0.92)
    mlflow.log_artifact("model.pt")
```

### 4. View Results

```bash
mlflow ui
# Open http://localhost:5000
```

### 5. Compare Runs

In the UI: Select 2 runs, click "Compare"

---

## Practical Project: Fine-Tuning Experiment Tracker

**Build:** Fine-tune a model with MLflow tracking.

Requirements:
- 5-10 different hyperparameter configurations
- Each logged as separate MLflow run
- Parameters: learning_rate, lora_r, lora_alpha, batch_size
- Metrics: training_loss, eval_loss, RAGAS faithfulness
- Artifacts: model adapter files, evaluation report
- Model Registry: Best model registered

**Deliverables:**
- training_script.py (with MLflow)
- compare_runs.py (script to find best)
- screenshot of MLflow UI showing all runs

---

## Debugging: 10 Common MLflow Issues

### 1. "No active run" Error

```python
# WRONG
mlflow.log_metric("loss", 0.5)  # No active run!

# RIGHT
with mlflow.start_run():
    mlflow.log_metric("loss", 0.5)
```

### 2. Metrics Not Appearing

```python
# Check run is active
print(mlflow.active_run())

# If None, start run first
with mlflow.start_run():
    ...
```

### 3. Artifact Upload Slow

```python
# Large files take time
# MLflow uploads to backend (S3, etc.)
# Can take minutes for 1GB model

# To speed up: log only important artifacts
mlflow.log_artifact("model.safetensors")  # Yes
mlflow.log_artifact("checkpoint_1000/")   # No (redundant)
```

### 4. Can't Find Artifact

```python
# Artifact path is relative to run
# Inside run dir:
with mlflow.start_run():
    # Creates: mlruns/0/run_id/artifacts/model.pt
    mlflow.log_artifact("model.pt")

# Load it
artifact_uri = mlflow.get_run(run_id).info.artifact_uri
model_path = f"{artifact_uri}/model.pt"
```

### 5. SQLite DB Corrupted

```bash
# MLflow uses SQLite by default
# Sometimes corrupts if process killed

# Fix: Backup and delete
mv mlflow.db mlflow.db.backup
# Start fresh
mlflow server --backend-store-uri sqlite:///mlflow.db
```

### 6. Remote Backend Connection Failed

```python
# Trying to use RDS but no connection
mlflow.set_tracking_uri(
    "postgresql://user:pass@rds.amazonaws.com/mlflow"
)

# Check credentials
import psycopg2
conn = psycopg2.connect("postgresql://...")  # Test connection
```

### 7. Metrics Appear as Nan

```python
# Logging non-numeric value
mlflow.log_metric("accuracy", "high")  # Wrong!
mlflow.log_metric("accuracy", 0.92)    # Right

# Check type before logging
assert isinstance(accuracy, (int, float))
```

### 8. No Checkpoints Saved

```python
# Model not saved
with mlflow.start_run():
    # Train model...
    
    # MUST save explicitly
    mlflow.pytorch.log_model(model, "model")
    # Or
    mlflow.transformers.log_model(model, "model")
```

### 9. Experiment Name Conflicts

```python
# Creating experiment that already exists
mlflow.set_experiment("my-exp")
mlflow.set_experiment("my-exp")  # Error!

# Fix: Use get_experiment_by_name first
exp = mlflow.get_experiment_by_name("my-exp")
if not exp:
    mlflow.set_experiment("my-exp")
```

### 10. Can't Load Model from Registry

```python
# Model URI wrong
mlflow.pytorch.load_model("models:/my-model/1")  # Wrong!
mlflow.pytorch.load_model("models:/my-model/production")  # Right

# Check registered models
import mlflow
client = mlflow.tracking.MlflowClient()
client.list_registered_models()
```

---

## Case Study: Uber ML Tracking

**Challenge:** 100s of ML models across Uber. Which models are in production? Which versions?

**Solution:** MLflow-based experiment tracking for all models. Every model must be registered with metrics, artifacts, owner.

**Result:** Complete visibility into all ML systems. Can rollback any model in seconds.

---

## 1-Week MLflow Checklist

- [ ] Day 1: Install MLflow, understand concepts
- [ ] Day 2: Log simple experiment manually
- [ ] Day 3: Integrate with HuggingFace Trainer
- [ ] Day 4: Compare 5 runs in UI
- [ ] Day 5: Register best model in Model Registry
- [ ] Day 6: Write script to auto-find best run
- [ ] Day 7: Set up MLflow server (local or remote)

---

## Resources

- [MLflow Documentation](https://mlflow.org/docs/)
- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)
- [MLflow Transformers Integration](https://mlflow.org/docs/latest/transformers/)

→ **[DVC: Data Versioning →](dvc.md)**
