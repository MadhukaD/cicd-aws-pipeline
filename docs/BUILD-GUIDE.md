# Build Guide: CI/CD Pipeline on AWS (with the "why" behind every step)

This is the learning journal for the project. The README explains **what** the project is. This file explains **how it was built, step by step, and why each decision was made**, including the mistakes that taught the most.

> Replace placeholders like `<ACCOUNT_ID>` with your own values. Never commit real account IDs, keys, or IPs into a public repo if you can avoid it.

---

## 0. The big picture

```
git push -> GitHub Actions -> lint -> test -> build image -> scan image -> push to ECR -> deploy to ECS Fargate
```

**Why this order? "Fail fast."** Cheap checks go first. Linting takes seconds, tests take a minute, a Docker build takes longer, and a deploy takes longest. A typo should fail in seconds, not after minutes of building.

**Tools used:** Python/Flask, pytest, Ruff, Docker, GitHub Actions, Trivy, AWS ECR, AWS IAM (OIDC), AWS ECS Fargate, CloudWatch.

**Roadmap followed:**

1. Repo and a tiny app
2. Dockerize
3. CI: lint and test
4. Build in CI and scan with Trivy
5. AWS setup: ECR and OIDC
6. Push to ECR from the pipeline
7. Run on ECS Fargate (manually first)
8. Automate the deploy
9. Polish: README, dependencies, Dependabot, lifecycle policy, cleanup

---

## 1. Repo and a tiny app

### Why a tiny app?
The pipeline is the star, not the app. A reviewer wants to see how you ship software, not how well you write Flask.

### Setup

```bash
mkdir cicd-aws-pipeline && cd cicd-aws-pipeline
git init
python -m venv .venv
source .venv/Scripts/activate      # Windows Git Bash. Linux/macOS: source .venv/bin/activate
pip install flask gunicorn
pip freeze > requirements.txt
```

- **Why a virtual environment?** It isolates this project's dependencies from the rest of the machine, avoiding "works on my laptop" bugs.
- **Why `pip freeze > requirements.txt`?** It records exact versions, so Docker and CI install the same thing. This is a *reproducible build*.

### `.gitignore` first, before anything else

```
.venv/
__pycache__/
.pytest_cache/
.env
*.tfstate
*.tfstate.*
.terraform/
```

**Why first?** Once a secret or junk folder is committed, it lives in git history even after you delete it. Prevention is far easier than cleanup. The Terraform lines are there because committing a `.tfstate` file (which can contain secrets) is a classic beginner mistake.

### The app (`app/main.py`)

```python
import os

from flask import Flask, jsonify

app = Flask(__name__)


@app.get("/")
def index():
    return jsonify(
        message="Hello from the CI/CD pipeline",
        version=os.getenv("APP_VERSION", "dev"),
    )


@app.get("/health")
def health():
    return jsonify(status="ok")
```

- **Why `/health`?** Load balancers and orchestrators constantly ask "are you alive?". Every production service has one.
- **Why `APP_VERSION` from an environment variable?** The pipeline injects the git commit SHA, so you can open the running app and see exactly which commit is deployed. Configuration through environment variables is a core idea of the [12-factor app](https://12factor.net/config).

### Tests (`tests/test_app.py`) and `pytest.ini`

```ini
[pytest]
pythonpath = .
```

**Why test such a trivial app?** A pipeline with nothing to check is just a script. Tests are the *gate*: if they fail, bad code never reaches production.

### Learning points
- **Run it locally first.** If you automate something broken, you debug the app and the pipeline at the same time.
- **Commit messages are part of your portfolio.** "Add Flask app with health endpoint and tests" tells a story. "update" tells nothing.

---

## 2. Dockerizing

### Why containers?
Your app works because *your laptop* has the right Python, packages, and OS. An **image** packages the app with everything it needs, so it runs identically on your laptop, a CI runner, and AWS. An *image* is the blueprint; a *container* is a running instance.

### Dockerfile

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN apt-get update \
    && apt-get upgrade -y --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN useradd --create-home appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

USER appuser

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--worker-class", "gthread", "--workers", "2", "--threads", "4", "--access-logfile", "-", "app.main:app"]
```

### Why each line

| Line | Reason |
|---|---|
| `python:3.12-slim` | Small base image: faster pulls and deploys, fewer packages that can have vulnerabilities |
| `PYTHONDONTWRITEBYTECODE=1` | No useless `.pyc` files in the container |
| `PYTHONUNBUFFERED=1` | Logs appear immediately instead of being buffered. Otherwise `docker logs` can look empty while the app is crashing |
| `apt-get upgrade` in **one** `RUN` with cleanup | Patches OS packages (added after Trivy found fixable vulnerabilities). Doing the cleanup in the same `RUN` means the package lists never persist in a layer |
| `useradd` + `USER appuser` | Containers run as root by default. A non-root user limits the damage if the app is exploited |
| `COPY requirements.txt` then `pip install`, **before** copying code | Docker caches layers. Dependencies change rarely, code changes constantly, so most rebuilds skip the slow install |
| `COPY app/ ./app/` | Only the app is copied: no tests, no `.git`, no venv |
| `EXPOSE 8000` | Documentation only. Ports are opened at run time |
| `CMD [...]` JSON form | Runs Gunicorn directly as the main process so it receives shutdown signals |
| `0.0.0.0` | Listen on all interfaces, so traffic from outside the container can reach the app. `127.0.0.1` would make it reachable only from inside |

### Why Gunicorn and not `flask run`?
Flask's built-in server is a development server and not meant for real traffic. Gunicorn is a production WSGI server.

### `.dockerignore`

```
.venv/
.git/
__pycache__/
.pytest_cache/
tests/
.env
*.tfstate*
.terraform/
```

**Why?** `docker build` first sends the whole folder (the *build context*) to the Docker engine. Without this file it would ship your venv, git history, and potentially secrets. It also speeds up builds.

### Build and run

```bash
docker build -t cicd-demo:local .
docker run --rm -p 8000:8000 -e APP_VERSION=local-test cicd-demo:local
```

- `-t name:tag` names the image. Tags identify versions. Later, tags are commit SHAs.
- `-p 8000:8000` maps a laptop port to a container port. Containers are isolated, so without it nothing outside can reach them.
- `--rm` deletes the container on exit. Without it, stopped containers pile up, and a reused `--name` fails with "name already in use".

### Learning points
- **Browse to `http://localhost:8000`, never `http://0.0.0.0:8000`.** `0.0.0.0` is a *bind address* meaning "all interfaces". It is not a place you can visit, and browsers show `ERR_ADDRESS_INVALID`.
- **Run it twice and watch `CACHED`.** That is the layer-ordering trick paying off.

---

## 3. CI with GitHub Actions

### Why CI?
1. **It catches "works on my machine."** The runner starts from nothing and installs from `requirements.txt`.
2. **It's a gate.** Later it blocks bad code from deploying.

**Vocabulary:** *workflow* = one YAML file in `.github/workflows/`; *job* = steps running on one fresh machine; *step* = one command or action; *runner* = the VM GitHub provides.

### Linting with Ruff

```bash
pip install ruff pytest
ruff check .          # finds rule violations
ruff check --fix .    # fixes fixable ones (like import order)
ruff format .         # fixes layout
```

- **`ruff format` and `ruff check --fix` are different jobs.** Format fixes layout. Check fixes rule violations such as import order (`I001`).
- **Rehearse the pipeline's commands locally** (`ruff check .`, `ruff format --check .`, `pytest`). If they pass on your laptop, they pass on the runner.

### The first workflow (`.github/workflows/ci.yml`, job 1)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: requirements*.txt

      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Lint
        run: ruff check .

      - name: Check formatting
        run: ruff format --check .

      - name: Test
        run: pytest
```

### Why each part

- **Triggers on push to `main` and on pull requests.** Checking a PR *before* merging is the point of a gate.
- **`permissions: contents: read`.** The `GITHUB_TOKEN` is restricted to read-only: the **principle of least privilege**. A compromised step can't push code.
- **`ubuntu-latest`.** A fresh Linux VM per run. This is why Windows `Scripts/` paths and Linux `bin/` paths differ between laptop and CI.
- **`actions/checkout`.** The runner starts empty. Forgetting this is a classic mistake.
- **`uses:` vs `run:`.** `uses` runs a reusable action. `run` runs a shell command.
- **`python-version: "3.12"` in quotes,** because YAML reads `3.10` as the number `3.1`. It also matches the Dockerfile, so we test on the Python we ship.
- **`cache: pip`** speeds up later runs.
- **`ruff format --check`** reports but never modifies. CI should verify, never silently change code.

### Dependencies: runtime vs dev

- `requirements.txt`: only what the app needs to **run** (Flask, Gunicorn and their dependencies).
- `requirements-dev.txt`: `-r requirements.txt` plus pytest and ruff.

**Why split?** Tools like linters and test runners don't belong in the production image. Smaller image, less for Trivy to scan.

Order matters when rebuilding them in a fresh venv:

```bash
pip install flask gunicorn
pip freeze > requirements.txt            # freeze BEFORE installing dev tools

pip install pytest ruff
echo "-r requirements.txt" > requirements-dev.txt          # single > overwrites
pip freeze | grep -iE "^(pytest|ruff)==" >> requirements-dev.txt   # >> appends
```

**Pin versions.** An unpinned linter can release new rules and turn the pipeline red with no code change.

### Break it on purpose
Change a test to fail, push, watch the red X, then fix with `git revert HEAD --no-edit`. **A pipeline you've never seen fail is a pipeline you don't trust.** `revert` adds a new commit instead of rewriting history, which is the safe habit on shared branches.

---

## 4. Build the image in CI and scan it with Trivy

### Why build in CI?
It proves the Dockerfile works from a clean checkout, and it produces the exact image to **scan** and later **push**. Scan what you ship.

### Supply-chain lesson: pin your actions
In March 2026, Trivy's GitHub Action and some of its releases were compromised: version tags were force-pushed to malicious code that stole credentials. The advisory lists safe versions (Trivy binary 0.69.2 and 0.69.3, `trivy-action` 0.35.0).

**What it teaches:**
- A tag like `@v4` is a label the owner can move. If the owner's account is compromised, your pipeline (which holds AWS access) runs the attacker's code.
- **Pin to a known-good version,** and **pin actions to a full 40-character commit SHA** taken from a trusted release. A SHA can't be moved.
- **Pin the tool too.** The action and the scanner binary are two separate things (`version: v0.69.3`).
- Never switch to `latest` just to silence an "update available" notice. Update deliberately: read release notes, check advisories, then bump.

### The job

```yaml
  build-and-scan:
    runs-on: ubuntu-latest
    needs: lint-and-test
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t cicd-demo:${{ github.sha }} .

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@<40-CHAR-SHA>  # v0.35.0
        with:
          version: v0.69.3
          image-ref: cicd-demo:${{ github.sha }}
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: "1"
```

- **`needs: lint-and-test`:** don't build an image of code that already failed.
- **`${{ github.sha }}` as the tag:** every image traces to the exact code that built it.
- **`severity: HIGH,CRITICAL`:** failing on everything drowns you in noise, and you start ignoring the scanner.
- **`ignore-unfixed: true`:** if no patch exists, failing the build can't help.
- **`exit-code: "1"`:** without it Trivy prints a report and exits green. A report is not a gate. Quoted so YAML doesn't read it as a number.

### Reading a Trivy result
Each row shows the library, CVE ID, severity, installed version, and **fixed version**. Our first failure was `zlib1g`, an **OS package** from the base image, not a Python dependency. Fix: patch OS packages in the Dockerfile (`apt-get upgrade`, in one `RUN` with cleanup).

**Don't trust the scanner blindly.** One finding had a title about a JavaScript package on a C library. Titles can be mismatched, so read the linked source. The fix was the same either way.

---

## 5. AWS setup: ECR and OIDC

### The problem
The obvious way to give a pipeline AWS access is an IAM user with a stored access key. Don't. Long-lived keys leak into logs, forks, and old laptops, and bots scan GitHub for them.

### OIDC in one picture
1. GitHub gives each workflow run a short-lived signed token: "I'm repo X, branch Y."
2. AWS has been told in advance to trust GitHub as an identity provider.
3. The workflow shows the token to AWS, which checks it and returns **temporary credentials** (about an hour).
4. No secret is stored anywhere, so nothing is left to leak.

### Account hygiene first
- MFA on the root user, and don't use root day to day.
- A **Budget alert**, because cloud mistakes show up on the bill. Set it before creating anything.
- Pick one region and stay in it. This project uses **`eu-north-1` (Stockholm)**. Resources are per-region, so "where did my repo go?" usually means the region selector.

### ECR repository
- Name `cicd-demo`, private.
- **Tag immutability: on.** A tag always means the same image, so nobody (or no buggy pipeline) can silently overwrite `abc123`. If you can't trace what's running, you can't debug or audit it. Trade-off: re-running the same commit can't push the same tag again.
- Scan on push: a second, independent check that also keeps scanning stored images as new CVEs appear.

### Register GitHub as an identity provider (once per AWS account)
- Type: OpenID Connect
- URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

### The IAM role and its trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<OWNER>@<OWNER_ID>/<REPO>@<REPO_ID>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

- **The `sub` condition is the real lock.** Without it, *any GitHub repo in the world* could assume your role, because they all get tokens from the same provider.
- **Use `StringEquals`, not `StringLike`.** `StringLike` allows wildcards, and a stray `*` quietly widens access.
- **Only `main` is trusted.** Pull-request runs can't touch AWS, so untrusted code never gets credentials.
- **Our token's `sub` included numeric IDs** (`MadhukaD@<id>/repo@<id>`). The classic documented format has none. IDs are permanent, while names can be reused, so the ID format closes a name-takeover gap. **Read the real value** in repo **Settings -> Actions -> OIDC**, or print it from a debug step, instead of guessing.

### Permissions policy: least privilege

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcrLogin",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "EcrPushToOneRepo",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "arn:aws:ecr:eu-north-1:<ACCOUNT_ID>:repository/cicd-demo"
    }
  ]
}
```

- **Trust says *who* can become the role. Permissions say *what it can do*.**
- `GetAuthorizationToken` needs `Resource: "*"` because it's an account-level login action. That's normal.
- Push actions are limited to one repository ARN. Avoid `AdministratorAccess` shortcuts.
- **The region in the ARN must match where the repo lives.** A wrong region gives a confusing "access denied".
- **The role ARN is not a secret.** It's useless without a token that passes the trust conditions. Store it in a repository **variable**, not a secret.

---

## 6. Push the image to ECR from the pipeline

### Design decision: same job
Each job runs on a fresh machine, so the image built in one job doesn't exist in another. Rebuilding would produce a *different* image from the one scanned. So the push steps live in the **same job, after the scan**. If Trivy fails, nothing is pushed.

### Repository variables (Settings -> Secrets and variables -> Actions -> Variables)
- `AWS_REGION` = `eu-north-1`
- `AWS_ROLE_ARN` = the role ARN

### Added to the `build-and-scan` job

```yaml
    permissions:
      contents: read
      id-token: write
    steps:
      # ... checkout, build, scan ...

      - name: Configure AWS credentials (OIDC)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        uses: aws-actions/configure-aws-credentials@<40-CHAR-SHA>  # vX.Y.Z
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: ecr-login
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        uses: aws-actions/amazon-ecr-login@<40-CHAR-SHA>  # vX.Y.Z

      - name: Tag and push image
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: |
          IMAGE="${{ steps.ecr-login.outputs.registry }}/cicd-demo:${{ github.sha }}"
          docker tag cicd-demo:${{ github.sha }} "$IMAGE"
          docker push "$IMAGE"
```

- **`id-token: write` lets the job *request* the OIDC token.** It does not grant write access to the repo. It's set on this job only, so the lint-and-test job can never talk to AWS.
- **The `if:` condition** restricts AWS steps to pushes to `main`. PRs still build and scan, but never touch AWS. It matches the trust policy.
- **`amazon-ecr-login`** gets a temporary Docker password and outputs the registry address, so the account ID isn't hardcoded.
- **`docker tag` then `docker push`:** Docker pushes to wherever the image *name* says. The same image gets a second label with the full ECR address.

---

## 7. Run the container on ECS Fargate (by hand first)

### Why by hand first?
Automation hides things. Clicking through once means the pipeline step later makes sense, and when something breaks you know which piece to inspect.

### Cost decision
An ALB bills hourly even when idle. For a learning project, the task runs with a **public IP restricted to my own IP**. The production design (ALB, private subnets, TLS) is the Terraform follow-up.

### The four ECS concepts

| Concept | What it is | Analogy |
|---|---|---|
| Cluster | A logical grouping for services | A folder |
| Task definition | Blueprint: image, CPU/memory, ports, env vars | A recipe |
| Task | One running copy of the recipe | A dish cooked from it |
| Service | Keeps N tasks running, replaces dead ones | A kitchen manager |

**Fargate** means AWS runs the servers. You say "0.25 vCPU, 0.5 GB" and never patch a machine.

### Settings used
- Cluster: `cicd-demo-cluster` (Fargate only)
- Task definition `cicd-demo`: Fargate, **Linux/X86_64**, 0.25 vCPU, 0.5 GB, container `app`, port 8000, env `APP_VERSION`, awslogs enabled
- Service `cicd-demo-service`: 1 desired task, default VPC, public IP **on**, security group `cicd-demo-sg` allowing TCP 8000 from **my public IP /32**

### Why these choices
- **X86_64:** GitHub's runners build x86 images. An architecture mismatch kills the container instantly with `exec format error`.
- **Execution role vs task role:** the *execution role* lets ECS itself pull from ECR and write logs, before your app starts. A *task role* is what your app's code uses to call AWS services. This app calls none, so there isn't one.
- **SHA tags, not `latest`:** a task definition that says `:abc123` always runs the same code. Rollback means pointing at the previous SHA.
- **CloudWatch logs:** containers are disposable. Logs must go somewhere durable.
- **Public IP on:** the task needs a route to ECR to pull the image. (Production uses private subnets with NAT or VPC endpoints.)
- **Security group = a firewall.** Opening a port to `0.0.0.0/0` gets scanned within minutes, so restrict it.

### Learning points from this step
- **A security group source must be CIDR**, such as `203.0.113.25/32`. `/32` means exactly one address. A bare IP is rejected as "malformed".
- **Use the public IP**, from `https://checkip.amazonaws.com`. `ipconfig` shows your private LAN address (`192.168.x.x`), which AWS can never see.
- **Many ISPs share one public IP (CGNAT) or change it often.** The rule may stop matching, so re-check it when the page times out.
- **Use `http://`, not `https://`.** No TLS is configured.
- **Each new task gets a new public IP.** ECS doesn't store the IP: the task has a network interface, and the IP belongs to it. This is exactly why real services use a load balancer with fixed DNS.

### The Gunicorn timeout lesson
The logs showed a repeating cycle: `WORKER TIMEOUT`, `Error handling request (no URI read)`, `Worker was sent SIGKILL! Perhaps out of memory?`.

- **"Out of memory?" was a red herring.** Gunicorn prints it for any SIGKILL. The real cause was its own 30-second timeout.
- **`no URI read` means a client opened a connection and sent nothing.** Browsers open speculative connections like this.
- **The default `sync` worker handles one connection at a time,** so one idle connection froze the whole app.
- **Fix:** `--worker-class gthread --workers 2 --threads 4`, plus `--access-logfile -`. A load balancer in front also absorbs this problem.
- **Gunicorn doesn't log requests by default.** Enable access logs, and send logs to stdout (`-`), never to files inside a container.

### Cost control
A running task plus a public IPv4 address costs a little every hour. When done for the day, set the service's **desired tasks to 0**. The configuration stays, and the cost stops.

---

## 8. Automate the deploy

### Goal
Replace a dozen console clicks with one job. Manual deploys are slow, easy to get wrong, and impossible to audit.

### Extra IAM permissions (second inline policy on the same role)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TaskDefinitions",
      "Effect": "Allow",
      "Action": ["ecs:DescribeTaskDefinition", "ecs:RegisterTaskDefinition"],
      "Resource": "*"
    },
    {
      "Sid": "UpdateOneService",
      "Effect": "Allow",
      "Action": ["ecs:UpdateService", "ecs:DescribeServices"],
      "Resource": "arn:aws:ecs:eu-north-1:<ACCOUNT_ID>:service/cicd-demo-cluster/cicd-demo-service"
    },
    {
      "Sid": "PassExecutionRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
      "Condition": {
        "StringEquals": { "iam:PassedToService": "ecs-tasks.amazonaws.com" }
      }
    }
  ]
}
```

- **Task-definition actions need `Resource: "*"`,** the accepted pattern.
- **`UpdateService` is limited to one service ARN.**
- **`iam:PassRole` exists to prevent privilege escalation.** Registering a task definition *hands* a role to ECS. Without a check, anyone who can register tasks could hand an admin role to a container. We allow passing only one specific role, only to ECS tasks. A missing `PassRole` is the most common cause of ECS deploy `AccessDenied` errors.
- **In production, teams often use separate roles for pushing and deploying,** to shrink the blast radius further.

### The deploy job

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: build-and-scan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    permissions:
      id-token: write
      contents: read
    env:
      ECS_CLUSTER: cicd-demo-cluster
      ECS_SERVICE: cicd-demo-service
      TASK_FAMILY: cicd-demo
    steps:
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@<40-CHAR-SHA>  # vX.Y.Z
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Register new task definition
        id: register
        run: |
          set -euo pipefail
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          IMAGE="${ACCOUNT_ID}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com/cicd-demo:${{ github.sha }}"

          aws ecs describe-task-definition \
            --task-definition "$TASK_FAMILY" \
            --query taskDefinition > current.json

          jq --arg img "$IMAGE" --arg ver "${{ github.sha }}" '
            del(.taskDefinitionArn, .revision, .status, .requiresAttributes,
                .compatibilities, .registeredAt, .registeredBy)
            | .containerDefinitions |= map(
                if .name == "app"
                then .image = $img | .environment = [{"name": "APP_VERSION", "value": $ver}]
                else . end)
          ' current.json > new.json

          NEW_ARN=$(aws ecs register-task-definition \
            --cli-input-json file://new.json \
            --query taskDefinition.taskDefinitionArn --output text)
          echo "Registered $NEW_ARN"
          echo "arn=$NEW_ARN" >> "$GITHUB_OUTPUT"

      - name: Update service and wait
        run: |
          aws ecs update-service \
            --cluster "$ECS_CLUSTER" \
            --service "$ECS_SERVICE" \
            --task-definition "${{ steps.register.outputs.arn }}" > /dev/null
          aws ecs wait services-stable \
            --cluster "$ECS_CLUSTER" \
            --services "$ECS_SERVICE"
```

### Why each part
- **`needs: build-and-scan`:** deploy only happens after the image was built, scanned, and pushed. This is the full gate chain.
- **Job-level `if`:** deploys only on pushes to `main`.
- **No `checkout` step:** the job only needs the AWS CLI, which is preinstalled. Fewer steps and one less third-party action.
- **Plain AWS CLI instead of another action:** after the Trivy incident, fewer third-party actions in a job that holds AWS access is a defensible choice.
- **Start from the current task definition,** so CPU, memory, ports, and log config carry over. `jq` changes only the image and `APP_VERSION`, and `del(...)` removes read-only fields AWS refuses to accept back.
- **Each deploy registers a new revision** (`cicd-demo:N+1`), so rollback is easy. Old revisions cost nothing.
- **`$GITHUB_OUTPUT`** passes the new revision's ARN to the next step.
- **`aws ecs wait services-stable`** blocks until the rollout finishes. Without it, the job goes green even if the new container crashes on boot.
- **`set -euo pipefail`** stops the script at the first error or unset variable.

### Verify end to end
Open `http://<task-public-ip>:8000/` and compare `version` with your latest commit SHA. A match proves the pipeline deployed exactly that code. `version: dev` means an old revision is running. `/health` has no version, so use `/`.

---

## 9. Polish

### Done
- **Split dependencies** into runtime and dev files (section 3).
- **README** with a Mermaid architecture diagram, CI status badge, stages table, security decisions, and "what I learned".

### Still to do
- **Dependabot:** proposes updates for pinned action SHAs, the Docker base image, and Python packages, so pinning doesn't mean going stale. This is the right answer to the "new Trivy version available" notice.
- **ECR lifecycle policy:** automatically delete old images so storage cost stays near zero.
- **Cleanup guide:** exactly what to delete to stop all costs (ECS service and cluster, task definition revisions, ECR images and repo, CloudWatch log group, security group, IAM role and OIDC provider if no longer needed).

---

## 10. Troubleshooting log: every real error and what it taught

| # | Symptom | Cause | Lesson |
|---|---|---|---|
| 1 | `source .venv/bin/activate: No such file` | On Windows, venv scripts live in `.venv/Scripts/`, not `bin/` | Laptop and CI runner are different OSes. Know the difference |
| 2 | `refusing to merge unrelated histories` | Two local repos: one created by `git init`, one cloned by GitHub Desktop | Keep one source of truth. Delete the other copy |
| 3 | `aws.exe: ... valid choices are get, store, erase` during `git push` | A leftover git `credential.helper` pointing at the AWS CLI | Harmless noise. Check with `git config --show-origin --get-all credential.helper` |
| 4 | Ruff `I001` import order | Imports not grouped per PEP 8 | `ruff check --fix` and `ruff format` do different jobs |
| 5 | Trivy red: `zlib1g` HIGH x2 | Base image not yet rebuilt with the OS patch | Patch OS packages in the Dockerfile. Read scanner output critically |
| 6 | `Unable to resolve action ... repository not found` | Typo in an action name | Copy names from the action's page. Don't type them |
| 7 | `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Trust policy `sub` didn't match the token's `sub` (which contained numeric IDs) | Look at both sides of the comparison. Read the real claim |
| 8 | `CIDR block ... is malformed` | Bare IP instead of CIDR, and a private LAN address | Use the public IP with `/32` |
| 9 | `WORKER TIMEOUT` and `no URI read` in logs | One idle connection blocked Gunicorn's single sync worker | Use `gthread`. "Out of memory?" was misleading |
| 10 | `ERR_ADDRESS_INVALID` | Browsed to `0.0.0.0` | Use `localhost` |
| 11 | Deploy job skipped | Typo `geads` instead of `heads` in the `if:` condition | A skipped job means its `if:` or a `needs` dependency was false |
| 12 | `argument --service is required` | Typed `-- service` with a space | A bare `--` ends option parsing. Paste instead of retyping |
| 13 | CI: `No module named 'flask'` | `requirements-dev.txt` didn't pull in `requirements.txt` | Test in a throwaway venv to behave like CI |

### General debugging habits that paid off
1. **Read the error literally.** It's usually accurate.
2. **Look at both sides of a comparison** (token claims vs trust policy, printed command vs intended command).
3. **Change one thing at a time,** and rehearse the pipeline's commands locally first.
4. **Paste, don't retype,** YAML, shell, and JSON.
5. **Check the region** before assuming something is missing in AWS.

---

## 11. Security decisions at a glance

- OIDC instead of stored access keys, with a trust policy limited to one repo and branch
- Least-privilege IAM: one ECR repo, one ECS service, one passable role
- Workflow `permissions` default to read-only. `id-token: write` only on jobs that need AWS
- Third-party actions pinned (SHA for scanner and AWS actions). First-party `actions/*` currently use version tags, a candidate for SHA pinning via Dependabot
- Image scan as a real gate (fails the build), and the deploy only runs after it passes
- Non-root container, slim base image, patched OS packages
- Immutable image tags, with the commit SHA as the tag
- Security group restricted to a single IP
- No secrets in the repo, and `.gitignore` and `.dockerignore` set up first

## 12. Where this project can go next

- **Terraform:** rebuild ECR, IAM, and ECS as code, with an ALB, private subnets, and TLS
- **Branch protection:** require CI checks before merging to `main`
- **Post-deploy smoke test:** `curl /` and fail the job if `version` doesn't match the commit
- **Automatic rollback** on a failed deployment
- **Scheduled rebuilds** so base-image patches are picked up even without code changes
- **`pip-tools` or `uv`** for fully locked dev dependencies