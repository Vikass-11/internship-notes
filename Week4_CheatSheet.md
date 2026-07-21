# Week 4 — Cheatsheet
**Internship Week 4 — DevOps & System Reliability**

---

## The big picture
```
Write code → push to GitHub → Actions runs lint/tests
    → builds Docker image → pushes to registry (DockerHub/ECR)
    → Terraform provisions cloud infra → EC2 pulls image via IAM role
    → container runs → app live
```
Everything below is one piece of this chain.

---

## Day 1 — CI/CD (GitHub Actions)

**CI** → automatically build and test every change.
**CD** → automatically deploy tested code.

### Hierarchy
```
Event → Workflow → Job → Step → Action/Run
```
- **Workflow** → the whole automation, lives in `.github/workflows/*.yml`
- **Job** → a chunk of work, runs on its own runner. Multiple jobs run in parallel by default; use `needs:` to make one wait for another.
- **Step** → one command/action inside a job, runs in order
- **Action** → prebuilt reusable step (`actions/checkout`, `actions/setup-node`)
- **Runner** → the VM that actually executes the job (`ubuntu-latest`)

### Common triggers
```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```
Other triggers: `workflow_dispatch` (manual button), `schedule` (cron)

### Real pattern I used — gate PRs, only ship on merge
```yaml
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '18' }
      - run: npm install
      - run: npx eslint src/
      - run: npm test

  build-and-push:
    needs: lint-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/devops-api:latest
```
First job runs on **every** PR/push. Second job only runs **after** the first passes, and only on a push to main — so a broken PR never gets a Docker image built for it.

---

## Day 2 — Cloud (AWS) + Terraform

### Why cloud
My laptop can't be on 24×7. Cloud = always-available infra.

### AWS basics I actually touched
- **EC2** → a VM, runs the app
- **ECR** → AWS's Docker registry (like DockerHub but inside AWS, integrates with IAM)
- **IAM** → who/what can do what
- **Security Group** → firewall rules for an instance (which ports are open)
- **Region** → `ap-south-1` (Mumbai) — pick the one closest to you

### Terraform = infra as code, language = HCL

### Flow
```
write main.tf → terraform init → terraform plan → terraform apply
```
- `init` → downloads the provider plugin (e.g. AWS), sets up the working dir
- `plan` → shows what *would* change, doesn't touch anything yet
- `apply` → actually creates/changes/destroys resources
- `destroy` → tears everything down

### Resource vs data block
```hcl
resource "aws_instance" "my_vm" { ... }   # creates something
data "aws_ami" "latest" { ... }           # looks something up, doesn't create
```

### Real resource chain I built
```
ECR repo → Security Group → IAM Role → IAM Policy → Instance Profile → EC2 instance
```
The instance profile is just the wrapper AWS needs to attach an IAM role to an EC2 instance — not a separate permission boundary itself.

### Gotchas I hit
- Changing `user_data` on an existing instance does **not** re-run the script — AWS only runs `user_data` on true first boot. Need `user_data_replace_on_change = true` or `terraform apply -replace="..."` to force a real fresh boot.
- `.terraform/` and `*.tfstate` should never go in git — `.terraform/` is just downloaded plugin binaries (huge), and `.tfstate` can contain sensitive resource data. `.terraform.lock.hcl` SHOULD be committed though (locks provider versions for the team).
- ECR repo won't delete via `terraform destroy` if it still has images in it — add `force_delete = true` to the repo resource.

---

## Day 3 — Security Best Practices

### Least privilege
Give only the minimum permissions needed. Nothing extra "just in case."

### What this looked like in practice (IAM policy for my EC2 role)
```hcl
# Allowed — and ONLY these:
ecr:GetAuthorizationToken     # required to even log in to ECR, must be account-wide
ecr:BatchGetImage             # read image metadata
ecr:GetDownloadUrlForLayer    # download the actual image layers

# Resource scoped to ONE specific repo ARN, not "*"
# NOT allowed: ecr:PutImage, ecr:DeleteRepository, any other AWS service
```
Justification matters more than the policy itself — for every permission, be able to say *why* it's there and what breaks without it.

### Secrets — never hardcode
```js
// Bad
password = "123456"

// Good
${{ secrets.DB_PASSWORD }}
```

**GitHub Secrets** → used during CI/CD (`DOCKERHUB_TOKEN`, `AWS_ACCESS_KEY_ID`, etc.)
**AWS Secrets Manager** → used by the running application itself, not the pipeline

The real distinction:
```
GitHub Secrets       → CI/CD pipeline only
AWS Secrets Manager   → app needs it at runtime
```

### Best practice for IAM users
- Never use root account day-to-day — create an IAM user instead
- Generate access keys via **CLI use case**, not console login, for tools like Terraform
- Rotate any key that gets pasted/exposed anywhere outside the AWS console, even accidentally

---

## Day 4 — Docker

### The problem it solves
"Works on my machine" — different OS, Node version, libraries between dev and prod. Docker packages everything together so it runs identically anywhere.

### VM vs Container
| Virtual Machine | Container |
|---|---|
| Has its own guest OS | Shares host OS kernel |
| Heavy | Lightweight |
| Slow startup | Fast startup |

### Lifecycle
```
Dockerfile → docker build → Docker Image → docker run → Container → App running
```
**Image** = blueprint, read-only, reusable. **Container** = a running instance of that image. One image → many containers.

### Dockerfile instructions
```dockerfile
FROM node:18-alpine        # base image
WORKDIR /app                # like `cd /app` inside the container
COPY package*.json ./       # copy dependency manifests first (caching!)
RUN npm install --omit=dev  # install deps, skip devDependencies
COPY . .                    # copy the rest of the source
EXPOSE 3000                  # documents the port (doesn't open it)
CMD ["node", "src/server.js"] # runs when container starts
```
Copying `package*.json` before the rest of the code is intentional — if dependencies haven't changed, Docker reuses the cached `npm install` layer instead of reinstalling every build.

### Commands I actually used
```bash
docker build -t devops-api .              # build image
docker run -p 3000:3000 devops-api        # run, map host:container port
docker ps                                  # running containers
docker ps -a                               # all containers, including stopped/failed
docker images                              # list images
docker tag <image> <registry-url>:<tag>   # tag for a registry
docker push <registry-url>:<tag>          # push to registry
docker login --username AWS --password-stdin <ecr-url>  # auth to ECR
```

### .dockerignore
Same idea as `.gitignore` — keep `node_modules`, `.git`, test files out of the build context.

---

## Kubernetes (concept only — didn't deploy this week)

Docker creates containers. Kubernetes manages many containers across many machines.

### Without K8s, manually:
restart crashed containers, scale, stop extras, decide placement.

### Hierarchy
```
Cluster → Node → Pod → Container
```
- **Cluster** → group of nodes
- **Node** → a machine (could be an EC2 instance)
- **Pod** → smallest deployable unit, usually one container

### What it automates
scheduling, restarting failed containers, autoscaling, load distribution, self-healing.

---

## IAM least-privilege — the actual exercise

This is the part I spent most time justifying, so worth its own section.

**Setup:** EC2 instance needs to pull a Docker image from ECR on boot. Nothing else.

**Role's trust policy** — who can assume this role:
```hcl
assume_role_policy = jsonencode({
  Statement = [{
    Action    = "sts:AssumeRole"
    Effect    = "Allow"
    Principal = { Service = "ec2.amazonaws.com" }
  }]
})
```
This just says "EC2 instances are allowed to use this role" — separate from what the role can actually *do*.

**What I deliberately did NOT grant:**
- No push access (can't publish images, only consume)
- No delete access on the repo
- No access to any other AWS service (S3, other EC2 actions, nothing)
- Resource scoped to one repo ARN, not `*`

**Why this matters:** if the instance is ever compromised, the attacker can read images from one repo. That's it. No lateral movement, no destructive capability.

---

## Failure simulation — what I actually broke and learned

**What I broke:** changed the image tag in `user_data` to a tag that was never pushed (`:v2-broken`).

**Symptom:** API totally unreachable, `docker ps -a` showed zero containers — not even a failed one.

**How I found the cause:**
1. SSH'd into the instance: `ssh -i key.pem ec2-user@<ip>`
2. Checked `/var/log/cloud-init-output.log` — this is where `user_data` script output lands on Amazon Linux
3. Found: `manifest for ...devops-api:v2-broken not found`
4. Confirmed Docker install and ECR login both succeeded (proved IAM role was correct) — isolated the problem to specifically the image tag
5. `set -e` in the script meant it exited the instant `docker pull` failed, so `docker run` never even ran — explains why there was no container at all

**What I'd do differently in a real setup:**
- Add a health check at the end of `user_data` that fails loudly if the container didn't start
- Ship `cloud-init-output.log` to CloudWatch Logs automatically instead of requiring manual SSH
- CloudWatch Alarm on instance health / a real HTTP health check
- Validate the image tag exists in the registry *before* triggering deployment, in CI
- Blue-green style rollback — keep the previous working version as fallback instead of overwriting in place

---

## Quick Ref
```
CI/CD:        Event → Workflow → Job → Step → Action | lint+test every PR, build only on merge
Terraform:    init → plan → apply → destroy | resource creates, data looks up
IAM:          least privilege | scope to one ARN | justify every permission
Secrets:      never hardcode | GitHub Secrets = CI/CD | AWS Secrets Manager = runtime
Docker:       FROM → WORKDIR → COPY → RUN → EXPOSE → CMD | image = blueprint, container = instance
Debugging:    SSH in → check cloud-init-output.log → isolate root cause → fix → verify
Kubernetes:   Cluster → Node → Pod → Container | automates restart/scale/heal
```
