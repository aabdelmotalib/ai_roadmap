# Fine-Tuning: Adapting Models to Your Domain

Fine-tuning isn't about adding knowledge. It's about changing behavior, style, or task-specific patterns.

A model trained on all of the internet doesn't know YOUR domain. Fine-tuning teaches it.

---

## Why This Matters

**Real incident:** Support bot trained on generic data. Didn't follow company guidelines. Cost: constant retraining, poor customer feedback.

**Real incident 2:** Company fine-tuned model, didn't validate. Model got worse (overfitting to small bad dataset). Deployed broken.

Fine-tuning is powerful but risky. You'll learn:
- When to fine-tune vs RAG vs prompt engineering
- How to prepare quality training data
- LoRA (Low-Rank Adaptation)
- Evaluation before/after

---

## Conceptual Explanation: Teaching a Skill

**Analogy:**

You hire a brilliant generalist who knows about everything.

Raw model = generalist
Fine-tuning = teaching them YOUR specific domain patterns

Example:
- Raw: "How should I format a support ticket?" → "Fill it out clearly"
- Fine-tuned on your data: "How should I format a support ticket?" → "Use Title | Priority | Category | Description fields. Always add environment info"

Fine-tuning doesn't add facts. It adds habits.

---

## Code Examples: LoRA Fine-Tuning

### Dataset Preparation

```python
import json
from datasets import Dataset

def prepare_training_data(examples: list[dict]) -> list[dict]:
    """Format data for fine-tuning."""
    formatted = []
    for ex in examples:
        formatted.append({
            "messages": [
                {"role": "system", "content": "You are a helpful customer support agent"},
                {"role": "user", "content": ex["question"]},
                {"role": "assistant", "content": ex["answer"]}
            ]
        })
    return formatted

# Save to JSONL (one JSON per line)
with open("training_data.jsonl", "w") as f:
    for ex in prepare_training_data(examples):
        f.write(json.dumps(ex) + "\n")

# Verify format
import dataset
ds = Dataset.from_json("training_data.jsonl")
print(f"Dataset size: {len(ds)} examples")
```

### LoRA Fine-Tuning with OpenAI API (Easiest)

```python
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def finetune_openai():
    """Create fine-tuning job."""
    
    # Upload training file
    with open("training_data.jsonl", "rb") as f:
        response = await client.files.create(
            file=f,
            purpose="fine-tune"
        )
        file_id = response.id
    
    # Create fine-tuning job
    job = await client.fine_tuning.jobs.create(
        training_file=file_id,
        model="gpt-3.5-turbo",  # Base model
        hyperparameters={
            "n_epochs": 3,  # Train for 3 passes
            "batch_size": 32,
            "learning_rate_multiplier": 1.0
        }
    )
    
    print(f"Job ID: {job.id}")
    # Wait for completion (usually 1-24 hours)
    
    # Use fine-tuned model
    response = await client.chat.completions.create(
        model=job.fine_tuned_model,  # my-model-2024-01-01
        messages=[{"role": "user", "content": "Help me with..."}]
    )
    return response.choices[0].message.content
```

### LoRA with HuggingFace (Open Source, Local)

```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments, Trainer
from datasets import load_dataset

# 1. Load base model (LLaMA 3 8B)
model_name = "meta-llama/Llama-2-7b-hf"
model = AutoModelForCausalLM.from_pretrained(model_name)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# 2. LoRA config (small adapters on top of base model)
lora_config = LoraConfig(
    r=8,  # Rank: how many parameters to adapt
    lora_alpha=16,  # Scaling factor
    target_modules=["q_proj", "v_proj"],  # Which layers to adapt
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

# 3. Wrap model with LoRA
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 1.4M (0.02% of 7B)

# 4. Prepare dataset
def tokenize_function(examples):
    return tokenizer(
        examples["text"],
        padding="max_length",
        max_length=512,
        truncation=True
    )

dataset = load_dataset("json", data_files="training_data.jsonl")
tokenized_dataset = dataset.map(tokenize_function, batched=True)

# 5. Training
training_args = TrainingArguments(
    output_dir="./lora_models",
    num_train_epochs=3,
    per_device_train_batch_size=4,  # Small GPU
    learning_rate=1e-4,
    save_steps=100,
    logging_steps=10
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset["train"]
)

trainer.train()

# 6. Save adapters (small file, ~10MB)
model.save_pretrained("my_lora_adapters")

# 7. Use fine-tuned model
from peft import AutoPeftModelForCausalLM

model = AutoPeftModelForCausalLM.from_pretrained("my_lora_adapters")
inputs = tokenizer("What is AI?", return_tensors="pt")
outputs = model.generate(**inputs, max_length=100)
print(tokenizer.decode(outputs[0]))
```

### QLoRA (4-bit Quantization, Very Cheap)

```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

# 4-bit quantization config
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)

# Load 13B model on T4 GPU (would normally need 26GB)
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-13b-hf",
    quantization_config=bnb_config,
    device_map="auto"
)

# Rest is same: LoRA on top of 4-bit model
# Memory: ~6GB instead of 26GB (4x improvement)
```

---

## Decision Framework: When to Fine-Tune

```
Do you have 100+ labeled examples?
  ├─ NO: Prompt engineering only
  └─ YES: Continue...

Is it behavioral (how to respond) vs knowledge (what to know)?
  ├─ KNOWLEDGE: Use RAG instead
  ├─ BEHAVIOR: Fine-tune acceptable
  └─ BOTH: RAG + fine-tune

Can you evaluate it?
  ├─ NO: Don't fine-tune (can't measure if it works)
  └─ YES: Fine-tune acceptable
```

---

## Step-by-Step: From Scratch to Fine-Tuned Model

### 1. Collect Data

```python
# 200 examples of your domain
training_data = [
    {
        "question": "How do I reset my password?",
        "answer": "Go to login page, click 'Forgot Password', verify email, set new password"
    },
    # ... 199 more examples
]
```

### 2. Format for Fine-Tuning

```python
formatted = prepare_training_data(training_data)
with open("training.jsonl", "w") as f:
    for ex in formatted:
        f.write(json.dumps(ex) + "\n")
```

### 3. Fine-Tune

```python
# Using OpenAI (easiest)
job = await client.fine_tuning.jobs.create(
    training_file=file_id,
    model="gpt-3.5-turbo"
)
# Wait 1-4 hours
```

### 4. Evaluate

```python
# Compare base vs fine-tuned
test_questions = [...]
for q in test_questions:
    base_answer = await call_base_model(q)
    ft_answer = await call_finetuned_model(q)
    # Which is better? Manual eval
```

### 5. Deploy

```python
# Use fine-tuned model in production
response = await client.chat.completions.create(
    model="ft:gpt-3.5-turbo:abc123",  # Your fine-tuned model
    messages=[...]
)
```

---

## Practical Project: Custom Support Bot

**Problem:** Generic support bot doesn't know company policies. Fine-tune on internal examples.

**Data:** 300 Q&A pairs (support tickets + ideal responses)

**Evaluate:** 50 test questions. Score: does fine-tuned bot follow company guidelines?

**Deploy:** Use fine-tuned model for customer support

---

## Debugging Playbook: Fine-Tuning Failures

### 1. Model Gets Worse (Overfitting)

**Symptom:** Training loss drops, but test loss increases.

**Cause:** Too few examples (100 is borderline). Model memorizes.

**Fix:**
```python
# Use more data (500+)
# Or reduce epochs (1 instead of 3)
# Or increase dropout
```

### 2. No Improvement After Fine-Tuning

**Symptom:** Fine-tuned ≈ base model. Wasted time.

**Cause:** Problem was knowledge, not behavior. Fine-tuning doesn't add facts.

**Fix:** Switch to RAG instead.

### 3. Training Too Slow

**Symptom:** 1 hour per epoch.

**Cause:** Large dataset, small model can't fit batch.

**Fix:** Use QLoRA. Or reduce dataset.

### 4. Out of Memory

**Symptom:** CUDA error.

**Cause:** Batch size too large for GPU.

**Fix:**
```python
# Reduce batch size
per_device_train_batch_size=4  # Was 32
```

### 5. Model Forgets Original Knowledge

**Symptom:** Fine-tuned model is good on fine-tuning data, bad on general questions.

**Cause:** Catastrophic forgetting. Updated too aggressively.

**Fix:**
```python
# Lower learning rate
learning_rate=1e-5  # Was 1e-4
# Or fewer epochs
num_train_epochs=1  # Was 3
```

---

## Common Mistakes

!!! danger "Mistake 1: Overfitting to Small Dataset"

50 examples of your domain. Fine-tune. Model memorizes. Test on new examples: garbage.

**Don't:** Fine-tune on <100 examples.
**Do:** Collect 500+.

!!! danger "Mistake 2: Fine-Tuning When RAG Needed"

You fine-tune for knowledge ("What's the latest policy?"). Base model doesn't know, fine-tuning adds only 200 examples.

**Don't:** Expect fine-tuning to teach facts.
**Do:** Use RAG for knowledge, fine-tuning for behavior.

!!! danger "Mistake 3: No Evaluation"

You fine-tune, deploy, users hate it.

**Don't:** Skip testing.
**Do:** Manually evaluate 50 examples before deploy.

!!! danger "Mistake 4: Merging Adapters Wrongly"

You merged LoRA adapters incorrectly. Model corrupted.

**Don't:** Manual merge.
**Do:** Use merge_and_unload() from peft library.

---

## Production Realism

**Tutorial:** Fine-tune on 100 examples, deploy.

**Production:**
- 500+ high-quality examples
- 80/10/10 train/val/test split
- Manual evaluation (compare base vs FT)
- A/B testing in production
- Revert plan if it fails

---

## Cost & Performance

**Fine-tuning costs:**
- OpenAI GPT-3.5-turbo: $0.08/1K training tokens (one-time)
- 500 examples × 200 tokens = 100K tokens = $8
- Open source: free (your GPU)

**Inference costs:**
- Fine-tuned model same rate as base (no premium)

**Speed:** No difference

---

## Security

**Training data:** Never include secrets, PII, or customer data in training.

**Adapter files:** LoRA adapters can be stolen. Secure them.

---

## Case Study: HubSpot

**Problem:** CRM has own domain language. Generic LLM doesn't understand.

**Solution:** Fine-tune GPT on 1000 support conversations.

**Result:** 20% better responses in domain, no degradation on general tasks.

---

## Interview Cheat Sheet

**Concept 1: LoRA**
*Q: What's LoRA?*
*A: Train small adapters instead of whole model. 1M params instead of 7B. Same quality, 7000x cheaper.*

**Concept 2: When Fine-Tune**
*Q: RAG or fine-tuning?*
*A: RAG for knowledge. Fine-tuning for behavior/style.*

**Concept 3: Overfitting**
*Q: Model gets worse on test data?*
*A: Overfitting. Training on examples, forgetting general knowledge.*

**Concept 4: QLoRA**
*Q: 4-bit quantization benefit?*
*A: Fine-tune large models on small GPUs. 13B model on T4 (6GB, not 26GB).*

**Concept 5: Evaluation**
*Q: How know if fine-tuning worked?*
*A: Compare base and fine-tuned on same test set. Manual scoring.*

---

## 10 Review Questions

1. LoRA adapters are 1% of base model size. Why so small?
2. Training data: 100 examples vs 500. Which is enough?
3. Fine-tune for knowledge vs behavior. Difference?
4. Epoch 1 loss: 1.2, Epoch 3 loss: 0.5. Good?
5. Test accuracy dropped after fine-tuning. Why?
6. QLoRA uses 4-bit quantization. Tradeoff?
7. Merge LoRA adapter into base model. Purpose?
8. You fine-tuned but model still bad. Debug first?
9. Fine-tuning cost $8. Inference cost?
10. 500 examples, took 2 hours to fine-tune. Reasonable?

---

## Flashcards

**Card 1: Fine-Tuning**
*Q: Definition?*
*A: Training a model on YOUR data to improve on specific tasks.*

**Card 2: LoRA**
*Q: What's trained in LoRA?*
*A: Small adapters (1M params), not whole model (7B params).*

**Card 3: Data Size**
*Q: Minimum training examples?*
*A: 100, but 500+ recommended.*

**Card 4: Epochs**
*Q: What's an epoch?*
*A: One pass through all training data.*

**Card 5: Batch Size**
*Q: Batch size 32 vs 4?*
*A: 32 is faster but needs more GPU memory. 4 fits on small GPU.*

**Card 6: Learning Rate**
*Q: High vs low learning rate?*
*A: High = fast training but risk overfitting. Low = stable but slow.*

**Card 7: Validation**
*Q: Why validation set?*
*A: Detect overfitting. If train loss drops but val loss rises = overfitting.*

**Card 8: Knowledge**
*Q: Fine-tuning adds knowledge?*
*A: No, adds behavior/style. For knowledge use RAG.*

**Card 9: Inference**
*Q: Fine-tuned model costs more to use?*
*A: No, same cost as base.*

**Card 10: Deployment**
*Q: Revert plan if fine-tune fails?*
*A: Yes, keep base model as fallback.*

---

## Teach-It-Back Prompt

**Explain fine-tuning to a stakeholder:**

"We have 500 support conversations. We want to fine-tune GPT to follow our style. Walk me through: data prep, training, evaluation, deployment."

**Model Answer:**

"We format 500 conversations as Q&A pairs in JSONL. Upload to OpenAI. Create fine-tuning job (3 epochs, LR 1e-4). Wait 4 hours. Model trains.

Evaluate: take 50 new questions (not in training). Compare base GPT vs our fine-tuned version. Does it follow company guidelines better? If yes, good. If no, fine-tuning failed (maybe too few examples).

If good: deploy fine-tuned model in production. Cost same as base GPT, but responses follow our style.

If bad: roll back to base. Maybe we need RAG instead (facts problem, not style)."

---

## 1-Week Checklist

- [ ] Collected 200+ training examples in your domain
- [ ] Formatted data as JSONL
- [ ] Created validation set (10% of data)
- [ ] Started fine-tuning job
- [ ] Monitored loss curves (should decrease)
- [ ] Created test set (50 examples)
- [ ] Evaluated base vs fine-tuned manually
- [ ] Fine-tuned model scored better on domain tasks
- [ ] Fine-tuned didn't degrade on general tasks
- [ ] Saved fine-tuned model weights
- [ ] Wrote deployment guide
- [ ] Tested locally before deploy

---

## Resources

- [OpenAI Fine-Tuning Guide](https://platform.openai.com/docs/guides/fine-tuning)
- [PEFT Library (LoRA)](https://github.com/huggingface/peft)
- [LLaMA Training (Alpaca)](https://github.com/tatsu-lab/stanford_alpaca)

---

## Next Page

You can customize models now.

Next: evaluation. Measure if your AI is actually good.

→ **[Evaluation & Benchmarking →](evaluation.md)**
