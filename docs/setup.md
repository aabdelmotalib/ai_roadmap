# Development Environment Setup Guide

This guide is for Linux developers. You'll get a production-grade development environment in about 1 hour.

---

## 1. Why Environment Setup Matters in Production

Skip this section if you want. But read the true story first.

### The $50,000 Mistake

A team at a fintech startup had a microservice that processed financial transactions. It worked perfectly on their laptops. They deployed it to AWS Lambda using Python 3.9.

Six months later, production broke. Not a code bug—a dependency version mismatch. The Lambda runtime was upgraded to Python 3.11, but the team's local environment was still 3.9. An obscure library (`dataclasses-json`) behaved differently. Transaction amounts got rounded incorrectly. Over two weeks, they lost $50,000 in misprocessed fees.

**The fix:** 30 minutes of environment pinning. Exact Python version. Exact dependency versions. Docker for consistency. They could have had it day one.

### What Actually Breaks

When developers skip proper environment setup:

- **Version mismatches** — Works on your machine, fails on the server
- **Missing credentials** — You accidentally commit API keys
- **Inconsistent dependencies** — `pip install` gives different results on different days
- **Hard-to-reproduce bugs** — "It works for me" becomes a meme on your team
- **Slow deployments** — No Docker means rebuilding everything from scratch each time
- **Security vulnerabilities** — Old versions of libraries with known exploits

This roadmap prevents all of that on day one.

---

## 2. System Requirements

### Minimum Requirements
- **RAM:** 16GB (8GB possible but painful)
- **Disk:** 50GB free (for Docker images, datasets, models)
- **CPU:** Any modern processor (Intel 4th gen+, AMD Ryzen)
- **OS:** Linux (Ubuntu 20.04+ recommended)

### Recommended (for GPU/fine-tuning work)
- **GPU:** NVIDIA RTX 3060 Ti or better; or use cloud GPUs (lambdalabs.com, vast.ai, cloud providers)
- **RAM:** 32GB
- **NVMe SSD:** 256GB+ (for model weights)

### Cloud Fallback
Don't have a powerful machine? Use:
- **Google Colab** — Free T4 GPU (12 hours/session)
- **AWS SageMaker** — Free tier includes some GPU compute
- **Vast.ai** — Rent GPU by the hour (~$0.30/hour for good hardware)

---

## 3. Complete Tool Installation

### 3.1 pyenv: Python Version Management

**What it is:** Lets you install and switch between Python versions. Critical because different projects need different Python versions, and the system Python is often outdated.

**Why we use it:** Production systems specify exact Python versions. You need the ability to test against 3.9, 3.10, 3.11, and 3.12. `pyenv` makes this trivial.

#### Installation

```bash
# Install dependencies
sudo apt-get update
sudo apt-get install -y build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev curl libncursesw5-dev \
  xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev

# Install pyenv
curl https://pyenv.run | bash

# Add to ~/.bashrc
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init --path)"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc

exec $SHELL
```

#### Install Python 3.11.7

```bash
# This takes 5-10 minutes on first run (compiles Python)
pyenv install 3.11.7

# Make it your default
pyenv global 3.11.7

# Verify
python --version  # Should print: Python 3.11.7
```

---

### 3.2 Poetry: Python Dependency Management

**What it is:** Replaces `pip` and `requirements.txt`. Creates `pyproject.toml` and `poetry.lock` for reproducible installs across machines.

**Why we use it:** Every project you build has dependencies. Poetry locks exact versions. A colleague clones your repo, runs `poetry install`, and gets *exactly* the same versions. No "works on my machine" arguments.

#### Installation

```bash
# Install poetry
curl -sSL https://install.python-poetry.org | python3 -

# Add to PATH
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# Verify
poetry --version
```

#### Configure Poetry

```bash
# This makes virtual environments live in the project folder
# (easier to manage, easier for Docker)
poetry config virtualenvs.in-project true

# Verify
poetry config --list | grep in-project
```

---

### 3.3 Docker

**What it is:** Containerization. Your app, its dependencies, and the OS all bundled together. Run on your laptop, AWS, Kubernetes—everywhere looks identical.

**Why we use it:** Phase 2 is all about Docker. You'll containerize every project. No Docker = can't deploy anything beyond Phase 1.

#### Installation

```bash
# Add Docker repository
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group (so you don't need sudo every time)
sudo usermod -aG docker $USER

# Apply group changes
newgrp docker

# Verify
docker --version
docker run hello-world
```

---

### 3.4 kubectl: Kubernetes CLI

**What it is:** Command-line tool for managing Kubernetes clusters. You'll use this in Phase 4 to deploy services.

**Why we use it:** Phase 4 assumes you can `kubectl apply` configurations to a cluster.

#### Installation

```bash
# Download latest kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make executable
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

kubectl version --client
```

---

### 3.5 AWS CLI v2

**What it is:** Command-line interface for AWS. Upload files, manage EC2 instances, configure S3 buckets without touching the console.

**Why we use it:** Phase 2 is on AWS. You'll use `aws s3 cp`, `aws ec2 describe-instances`, `aws iam`, etc.

#### Installation

```bash
# Download and extract
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
```

#### Configure AWS

```bash
# This prompts for your AWS credentials (get from AWS Console)
aws configure

# You'll be asked for:
# AWS Access Key ID: [paste your key]
# AWS Secret Access Key: [paste your secret]
# Default region: us-east-1
# Default output format: json

# Verify
aws sts get-caller-identity  # Should print your AWS account info
```

---

### 3.6 Git

**What it is:** Version control. You already use this, but we need to configure it properly.

**Why we use it:** Every project in this roadmap lives in a Git repo.

#### Installation

```bash
sudo apt-get install git
git --version
```

#### Configure Git

```bash
# Set your identity (appears in every commit)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default branch (GitHub moved away from 'master')
git config --global init.defaultBranch main

# Set up SSH (so you don't type passwords)
ssh-keygen -t ed25519 -C "your.email@example.com"  # Press Enter 3x to skip passphrase
cat ~/.ssh/id_ed25519.pub  # Copy this output

# Paste the key into GitHub: Settings → SSH and GPG Keys → New SSH Key
# Test the connection:
ssh -T git@github.com  # Should print: "Hi [username]! You've successfully authenticated"
```

---

### 3.7 httpie: Modern curl Alternative

**What it is:** Makes testing APIs super easy. `curl` is powerful but verbose. `httpie` is beautiful.

**Why we use it:** Phase 1 you'll be testing APIs constantly. `http POST localhost:8000/chat message="hello"` is way better than `curl -X POST ...`.

#### Installation

```bash
sudo apt-get install httpie
http --version
```

---

### 3.8 jq: JSON Processor

**What it is:** Extracts and transforms JSON on the command line.

**Why we use it:** LLM APIs return nested JSON. `jq` extracts exact fields you need.

#### Installation

```bash
sudo apt-get install jq
jq --version
```

#### Example Usage

```bash
# Extract a field from JSON
echo '{"message": "hello", "status": 200}' | jq '.message'
# Output: "hello"

# Format API response
http https://api.example.com/data | jq '.results[] | {id, name}'
```

---

### 3.9 tmux: Terminal Multiplexer

**What it is:** Run multiple terminal sessions in one window. Long training jobs run in the background while you work.

**Why we use it:** Phase 3+ you'll have long-running jobs. `tmux` keeps them alive even if your terminal closes.

#### Installation

```bash
sudo apt-get install tmux
tmux --version
```

#### Quick Usage

```bash
# Start a new session
tmux new-session -s train

# Now you're in tmux. Run your training:
python train.py

# Detach (leave training running): Ctrl+B then D
# List sessions
tmux list-sessions

# Reconnect
tmux attach-session -t train

# Kill session
tmux kill-session -t train
```

---

## 4. VS Code Setup

### 4.1 Essential Extensions

Open VS Code → Extensions (Ctrl+Shift+X) → Search and install:

| Extension ID | Install |
|--------------|---------|
| `ms-python.python` | Python extension (debugging, linting) |
| `ms-python.pylance` | Fast type checking and IntelliSense |
| `ms-python.black-formatter` | Code formatting (Black) |
| `charliermarsh.ruff` | Ultra-fast linting (Ruff) |
| `ms-azuretools.vscode-docker` | Docker management from VS Code |
| `eamodio.gitlens` | Git integration (blame, history) |
| `rangav.vscode-thunder-client` | API testing (like Postman) |
| `humao.rest-client` | Send HTTP requests from `.http` files |
| `ms-toolsai.jupyter` | Jupyter notebook support |

### 4.2 settings.json Configuration

Open Command Palette (Ctrl+Shift+P) → "Preferences: Open Settings (JSON)" → Paste this:

```json
{
  // Python environment
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.linting.enabled": true,
  
  // Ruff linting (fast, comprehensive)
  "python.linting.ruffEnabled": true,
  "python.linting.ruffArgs": ["--line-length=100"],
  
  // Type checking with Pylance
  "python.analysis.typeCheckingMode": "strict",
  "[python]": {
    // Black formatter on save
    "editor.defaultFormatter": "ms-python.black-formatter",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  },
  
  // Formatting options
  "editor.rulers": [100],  // Visual line at 100 chars
  "editor.trimAutoWhitespace": true,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  
  // Git
  "[git-commit]": {
    "editor.rulers": [50, 72]  // Enforce commit message length
  },
  
  // Markdown
  "[markdown]": {
    "editor.wordWrap": "on",
    "editor.insertSpaces": true,
    "editor.tabSize": 2
  },
  
  // General
  "editor.fontSize": 12,
  "editor.fontFamily": "'Monaco', 'Courier New', monospace",
  "editor.wordWrap": "off",
  "editor.insertSpaces": true,
  "editor.tabSize": 4,
  "files.exclude": {
    "**/__pycache__": true,
    "**/.pytest_cache": true,
    "**/.mypy_cache": true,
    "**/.ruff_cache": true,
    "**/.venv": true
  },
  "search.exclude": {
    "**/.venv": true,
    "**/node_modules": true
  }
}
```

---

## 5. Complete .env Template

Create a `.env.example` file in every project. This shows what environment variables are needed without exposing secrets.

```bash
# Create the template
cat > .env.example << 'EOF'
# ============================================
# LLM APIs
# ============================================
OPENAI_API_KEY=your_key_here
OPENAI_MODEL=gpt-4-turbo
ANTHROPIC_API_KEY=your_key_here
COHERE_API_KEY=your_key_here
HUGGINGFACE_TOKEN=your_token_here

# ============================================
# Vector Databases
# ============================================
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=your_key_here
PINECONE_API_KEY=your_key_here
PINECONE_ENV=gcp-starter

# ============================================
# Relational Databases
# ============================================
# Format: postgresql+asyncpg://user:password@host:port/database
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/aiapp
MONGODB_URI=mongodb+srv://user:password@cluster.mongodb.net/dbname
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# ============================================
# AWS
# ============================================
AWS_ACCESS_KEY_ID=your_key_here
AWS_SECRET_ACCESS_KEY=your_secret_here
AWS_DEFAULT_REGION=us-east-1
AWS_ACCOUNT_ID=123456789012
S3_BUCKET_NAME=my-ai-bucket
S3_REGION=us-east-1

# ============================================
# MLOps Tools
# ============================================
MLFLOW_TRACKING_URI=http://localhost:5000
MLFLOW_EXPERIMENT_NAME=default
WANDB_API_KEY=your_key_here
WANDB_PROJECT=ai-roadmap
WANDB_ENTITY=your-username

# ============================================
# Application Configuration
# ============================================
APP_ENV=development
APP_NAME=ai_full_stack
APP_VERSION=0.1.0
SECRET_KEY=your-secret-key-change-in-production
DEBUG=true
LOG_LEVEL=INFO

# ============================================
# Service Endpoints (Development)
# ============================================
API_BASE_URL=http://localhost:8000
DATABASE_ECHO=false  # Set true to see SQL queries in logs
EOF

# See the example (don't check it into git!)
cat .env.example

# Create the actual .env (git will ignore this)
cp .env.example .env
```

Then in your code:

```python
from dotenv import load_dotenv
import os

# Load from .env file
load_dotenv()

# Access variables
api_key = os.getenv("OPENAI_API_KEY")
db_url = os.getenv("DATABASE_URL")
```

**Critical:** Add to `.gitignore`:

```
.env
.env.local
.env.*.local
```

Never commit real secrets. Ever.

---

## 6. Free Credits & GPU Access Guide

You're not paying $1000/month for cloud compute. Here's what's free:

### LLM APIs

| Provider | Free Tier | How to Claim |
|----------|-----------|------------|
| **OpenAI** | $5 credit (new accounts only) | Sign up at openai.com; credit expires after 3 months |
| **Anthropic** | Apply for free tier access | [Apply here](https://www.anthropic.com/earlyaccess); get $100 credits |
| **HuggingFace** | Free inference API | No credit card needed; limited requests/day |
| **Google Colab** | Free T4 GPU (12 hours/session) | colab.research.google.com; runs Jupyter notebooks |
| **Groq** | Free API (very fast) | [Free tier](https://console.groq.com); limited requests |

### Cloud Compute

| Provider | Free Tier | How to Claim |
|----------|-----------|------------|
| **AWS** | 12 months free tier (includes EC2, S3, RDS) | [Free tier](https://aws.amazon.com/free/); credit card required; track usage carefully |
| **Google Cloud** | $300 credit (90 days) | [Free tier](https://cloud.google.com/free); credit card required |
| **Azure** | $200 credit (30 days) | [Free tier](https://azure.microsoft.com/free); credit card required |
| **Heroku** | Discontinued free tier; use Railway or Fly.io instead | Railway.app offers free tier; Fly.io offers $3 credit |

### GPU Access (for fine-tuning)

| Provider | Cost | Use Case |
|----------|------|----------|
| **Google Colab Pro** | $10/month | Best value; T4 GPU (up to 8 hours), TPU (limited) |
| **Kaggle** | Free | 30 hours/week free GPU; no guarantee on type |
| **Vast.ai** | ~$0.30-0.60/hour | Rent consumer GPUs; cheapest for small jobs |
| **Lambda Labs** | ~$0.29-0.38/hour | Professional GPUs (RTX 6000); better for serious work |
| **RunPod** | ~$0.07-0.40/hour | Spot pricing (unreliable but dirt cheap) |

### Recommendation for This Roadmap

**Phase 0-2:** Stick with free tier. No fine-tuning needed yet.

**Phase 1 (fine-tuning arrives):** Use Google Colab Pro ($10/month) or Kaggle (free).

**Phase 3-4:** Get a $10/month Colab Pro subscription or open an AWS account (track bills closely).

---

## 7. GitHub Setup

### 7.1 SSH Key Configuration

```bash
# Generate SSH key (nothing is secret; use email)
ssh-keygen -t ed25519 -C "your.email@example.com"

# Accept defaults (just press Enter 3 times)
# This creates ~/.ssh/id_ed25519 and ~/.ssh/id_ed25519.pub

# Copy the public key
cat ~/.ssh/id_ed25519.pub

# Go to GitHub.com → Settings → SSH and GPG Keys → New SSH Key
# Paste the output from above

# Test the connection
ssh -T git@github.com
# Expected: "Hi [your-username]! You've successfully authenticated, but GitHub does not provide shell access."
```

### 7.2 First Repository Structure

```bash
# Create the project directory
mkdir my-ai-project
cd my-ai-project
git init

# Create the structure
mkdir -p src/myapp tests docs notebooks data models

# Create core files
touch README.md pyproject.toml Makefile .gitignore .pre-commit-config.yaml
```

### 7.3 .gitignore Template

```bash
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
ENV/
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/
.hypothesis/

# Type checking
.mypy_cache/
.dmypy.json
dmypy.json
.pyre/
.ruff_cache/

# IDEs
.vscode/
.idea/
*.swp
*.swo
*~

# Environment
.env
.env.local
.env.*.local

# Data & Models (don't commit large files)
data/raw/
data/processed/
models/
*.pickle
*.pkl
*.joblib

# OS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Misc
.cache/
tmp/
temp/
EOF

cat .gitignore
```

### 7.4 Create GitHub Repository

```bash
# Option A: Using GitHub CLI (easiest)
apt-get install gh
gh auth login  # Follow prompts
gh repo create my-ai-project --public --source=. --remote=origin --push

# Option B: Manual (via GitHub web)
# 1. Go to GitHub.com → New repository
# 2. Name: my-ai-project
# 3. Public (so portfolio is visible)
# 4. Don't initialize with README (you have one)
# 5. Copy the SSH URL
git remote add origin git@github.com:YOUR-USERNAME/my-ai-project.git
git branch -M main
git push -u origin main
```

---

## 8. Makefile Template

Create a `Makefile` in your project root. This centralizes common commands.

```makefile
# AI Full Stack Roadmap Project Makefile
# Run: make [target]

.PHONY: help install serve test lint format typecheck security docker-build docker-run clean

# Default target
help:
	@echo "Available targets:"
	@echo "  make install        - Install dependencies with poetry"
	@echo "  make serve          - Run FastAPI server (auto-reload)"
	@echo "  make test           - Run tests with coverage"
	@echo "  make lint           - Check code style (ruff) and formatting (black)"
	@echo "  make format         - Format code (black) and fix linting (ruff)"
	@echo "  make typecheck      - Type checking with mypy"
	@echo "  make security       - Security scan with bandit"
	@echo "  make docker-build   - Build Docker image"
	@echo "  make docker-run     - Run Docker container locally"
	@echo "  make deploy         - Deploy to production (placeholder)"
	@echo "  make clean          - Remove cache and build artifacts"

# Install project dependencies
install:
	poetry install

# Run the FastAPI development server
# uvicorn auto-reloads on file changes
serve:
	poetry run uvicorn src.myapp.main:app --reload --host 0.0.0.0 --port 8000

# Run tests with coverage report
test:
	poetry run pytest tests/ -v --cov=src --cov-report=html --cov-report=term-missing

# Lint code (check style without modifying)
lint:
	poetry run ruff check src/ tests/
	poetry run black --check src/ tests/

# Format code (actually modify files)
format:
	poetry run black src/ tests/
	poetry run ruff check --fix src/ tests/

# Type checking with mypy
typecheck:
	poetry run mypy src/

# Security scanning with bandit
security:
	poetry run bandit -r src/ --level medium

# Build Docker image
docker-build:
	docker build -t my-ai-project:latest .

# Run Docker container locally
docker-run:
	docker run --env-file .env -p 8000:8000 my-ai-project:latest

# Deploy placeholder (you'll fill in your deployment script)
deploy:
	@echo "Deployment script not yet configured"
	@echo "Add your deployment commands here"

# Clean up temporary files
clean:
	find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name .pytest_cache -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name .mypy_cache -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name .ruff_cache -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name htmlcov -exec rm -rf {} + 2>/dev/null || true
	find . -type f -name "*.pyc" -delete
	find . -type f -name ".coverage" -delete
	rm -rf dist/ build/ *.egg-info/

.DEFAULT_GOAL := help
```

Usage:

```bash
make help          # List all targets
make install       # Install dependencies
make serve         # Start dev server
make format        # Format code
make test          # Run tests
```

---

## 9. Pre-commit Hooks Setup

Pre-commit hooks automatically check your code before each commit. Catch bugs locally instead of in CI.

### 9.1 Create .pre-commit-config.yaml

```yaml
# .pre-commit-config.yaml
# Install: pre-commit install
# Uninstall: pre-commit uninstall

repos:
  # Code formatting (Black)
  - repo: https://github.com/psf/black
    rev: 24.1.1
    hooks:
      - id: black
        language_version: python3.11
        args: [--line-length=100]

  # Fast linting (Ruff)
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.11
    hooks:
      - id: ruff
        args: [--fix, --line-length=100]
      - id: ruff-format

  # Type checking (MyPy)
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
        args: [--strict]

  # Security scanning (Bandit)
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.5
    hooks:
      - id: bandit
        args: [--level, medium]

  # General file checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: mixed-line-ending
      - id: debug-statements
```

### 9.2 Install Pre-commit

```bash
# Install pre-commit package
poetry add --group dev pre-commit

# Install the git hooks
pre-commit install

# Test it (runs on all files)
pre-commit run --all-files

# Now, pre-commit runs automatically on git commit
# To skip (rarely): git commit --no-verify
```

---

## 10. Verify Your Setup

Create `verify_setup.py` in your project root. Run it to check everything.

```python
#!/usr/bin/env python3
"""
Verify that all required tools and dependencies are installed.
Usage: python verify_setup.py
"""

import os
import sys
import subprocess
from pathlib import Path
from dotenv import load_dotenv

# Colors for output
GREEN = "\033[92m"
RED = "\033[91m"
YELLOW = "\033[93m"
RESET = "\033[0m"

def check_command(cmd: str, version_arg="--version") -> bool:
    """Check if a command exists and is executable."""
    try:
        result = subprocess.run(
            [cmd, version_arg],
            capture_output=True,
            timeout=5
        )
        return result.returncode == 0
    except (FileNotFoundError, subprocess.TimeoutExpired):
        return False

def check_file(path: str) -> bool:
    """Check if a file exists."""
    return Path(path).exists()

def check_env_var(var_name: str) -> bool:
    """Check if an environment variable is set."""
    return os.getenv(var_name) is not None

def print_result(check_name: str, passed: bool, details: str = ""):
    """Print a check result with color."""
    status = f"{GREEN}✓{RESET}" if passed else f"{RED}✗{RESET}"
    message = f"{status} {check_name}"
    if details:
        message += f" ({details})"
    print(message)

def main():
    """Run all verification checks."""
    print("\n" + "="*60)
    print("🔍 AI Full Stack Engineer Setup Verification")
    print("="*60 + "\n")
    
    checks_passed = 0
    checks_failed = 0
    
    # System Tools
    print(f"{YELLOW}📦 System Tools{RESET}")
    tools = ["python", "git", "docker", "kubectl", "aws", "poetry", "tmux"]
    for tool in tools:
        passed = check_command(tool)
        print_result(tool.capitalize(), passed)
        if passed:
            checks_passed += 1
        else:
            checks_failed += 1
    
    # Python Version
    print(f"\n{YELLOW}🐍 Python Environment{RESET}")
    try:
        result = subprocess.run(
            ["python", "--version"],
            capture_output=True,
            text=True
        )
        version = result.stdout.strip()
        passed = "3.11" in version
        print_result("Python version", passed, version)
        if passed:
            checks_passed += 1
        else:
            checks_failed += 1
    except Exception:
        print_result("Python version", False, "Could not determine version")
        checks_failed += 1
    
    # Project files
    print(f"\n{YELLOW}📁 Project Structure{RESET}")
    required_files = [
        ("pyproject.toml", "Poetry configuration"),
        (".gitignore", "Git ignore file"),
        (".env.example", "Environment template"),
        ("Makefile", "Build automation"),
        (".pre-commit-config.yaml", "Pre-commit hooks"),
    ]
    
    for file_path, description in required_files:
        passed = check_file(file_path)
        print_result(description, passed, f"({file_path})")
        if passed:
            checks_passed += 1
        else:
            checks_failed += 1
    
    # Environment variables
    print(f"\n{YELLOW}🔐 Environment Variables{RESET}")
    load_dotenv()  # Load from .env file
    
    env_vars = [
        "OPENAI_API_KEY",
        "DATABASE_URL",
        "REDIS_URL",
        "AWS_ACCESS_KEY_ID",
    ]
    
    for var in env_vars:
        passed = check_env_var(var)
        status = "configured" if passed else "not configured"
        print_result(f"{var}", passed, status)
        if passed:
            checks_passed += 1
        else:
            checks_failed += 1
    
    # Summary
    print("\n" + "="*60)
    print(f"{YELLOW}Summary{RESET}")
    print(f"Checks passed: {GREEN}{checks_passed}{RESET}")
    print(f"Checks failed: {RED}{checks_failed}{RESET}")
    
    if checks_failed == 0:
        print(f"\n{GREEN}✓ Your environment is ready!{RESET}")
        print("Run: make help")
        return 0
    else:
        print(f"\n{RED}✗ Fix the above issues before proceeding{RESET}")
        return 1

if __name__ == "__main__":
    sys.exit(main())
```

Run it:

```bash
python verify_setup.py
```

Add to Makefile:

```makefile
verify:
	poetry run python verify_setup.py
```

---

## 11. Debugging Setup Issues

### Issue 1: `pyenv: command not found`

**Problem:** `pyenv` installation didn't work.

**Fix:**
```bash
# Check if it's installed
which pyenv

# If not found, reinstall
curl https://pyenv.run | bash

# Make sure it's in PATH (check ~/.bashrc)
echo 'export PATH="$HOME/.pyenv/bin:$PATH"' >> ~/.bashrc
exec $SHELL
```

---

### Issue 2: `poetry: command not found`

**Problem:** Poetry is installed but not in PATH.

**Fix:**
```bash
# Check installation
python3 -m pip show poetry

# Add to PATH
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
exec $SHELL

# Reinstall if needed
curl -sSL https://install.python-poetry.org | python3 -
```

---

### Issue 3: Docker daemon not running

**Problem:** `docker run` fails with "Cannot connect to Docker daemon"

**Fix:**
```bash
# Start Docker daemon
sudo systemctl start docker
sudo systemctl enable docker  # Start on boot
```

---

### Issue 4: `aws configure` fails to save credentials

**Problem:** AWS CLI won't authenticate.

**Fix:**
```bash
# Check if credentials are saved
cat ~/.aws/credentials
cat ~/.aws/config

# Reconfigure
aws configure

# Test
aws sts get-caller-identity
```

---

### Issue 5: Git SSH key not working

**Problem:** `ssh -T git@github.com` fails.

**Fix:**
```bash
# Generate key (if not already done)
ssh-keygen -t ed25519 -C "your.email@example.com"

# Copy public key to GitHub
cat ~/.ssh/id_ed25519.pub  # Copy this

# Go to: GitHub Settings → SSH and GPG Keys → New SSH Key → Paste

# Test
ssh -T git@github.com
# Expected: "Hi [username]! You've successfully authenticated..."
```

---

### Issue 6: VS Code can't find Python interpreter

**Problem:** Red squiggles everywhere, IntelliSense doesn't work.

**Fix:**
```bash
# In VS Code, open Command Palette (Ctrl+Shift+P)
# Type: "Python: Select Interpreter"
# Choose: "./venv/bin/python" or the pyenv version

# Or manually set in settings.json:
"python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python"
```

---

### Issue 7: Pre-commit hook fails on commit

**Problem:** `git commit` fails because pre-commit hooks found issues.

**Fix:**
```bash
# This is actually good! It caught issues.
# See the output to fix code

# Then try again
git add -A
git commit -m "Fix linting issues"

# If you really want to skip (rarely):
git commit --no-verify
```

---

### Issue 8: poetry.lock conflicts

**Problem:** Multiple developers add different dependencies; lockfile is messy.

**Fix:**
```bash
# Update and clean lock
poetry update

# If poetry install fails
rm poetry.lock
poetry install
```

---

### Issue 9: `DATABASE_URL` not recognized

**Problem:** Code can't connect to database.

**Fix:**
```bash
# Check .env file exists
ls -la .env

# Verify DATABASE_URL is there
cat .env | grep DATABASE_URL

# Test manually
poetry run python -c "import os; from dotenv import load_dotenv; load_dotenv(); print(os.getenv('DATABASE_URL'))"
```

---

### Issue 10: Port 8000 already in use

**Problem:** `make serve` fails; "Address already in use"

**Fix:**
```bash
# Find process using port 8000
lsof -iTCP:8000

# Kill the process
kill -9 <PID>

# Or use a different port
poetry run uvicorn src.myapp.main:app --reload --port 8001
```

---

## 12. Next Steps

**Your environment is ready.** Here's what's next:

1. **Create your first project repository** (follow section 7 above)
2. **Initialize Poetry** (run `poetry init` in the project directory)
3. **Set up pre-commit hooks** (section 9 above)
4. **Read Phase 0 Overview** — Start building
5. **Join the community** — Slack, Discord, or GitHub Discussions

You're 1 hour away from writing production-grade AI code.

Let's go. [Phase 0 Overview →](../phase0/overview.md)
