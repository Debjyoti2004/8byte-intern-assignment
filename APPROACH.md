# Approach & Design Decisions

This document explains the methodology, design choices, and implementation approach for the 8byte DevOps intern assignment.

---

## 1. Overall Methodology

The work was done in four phases:

1. **Application & local run** — Implement a minimal Node.js app and ensure it runs locally.
2. **Containerization** — Package the app in Docker for consistent, portable deployment.
3. **Infrastructure as Code** — Use Terraform to create a VPC, subnet, security group, and EC2 instance with Docker pre-installed.
4. **CI/CD** — Automate build and push of the Docker image to Docker Hub on every push to `master`.

Each phase was validated before moving to the next (local → Docker → Terraform → EC2 deploy → pipeline).

---

## 2. Application Design

- **Stack:** Node.js 18 + Express for a simple HTTP server on port 3000.
- **Scope:** Single route (`/`) returning a success message to satisfy the assignment and allow quick verification in the browser.
- **Rationale:** Keeps the focus on DevOps (Docker, Terraform, CI/CD) rather than application complexity.

---

## 3. Docker Strategy

### Multi-stage build

- **Stage 1 (builder):** Installs dependencies with `npm ci` and holds app code. Uses `node:18-alpine` for smaller footprint.
- **Stage 2 (runner):** Copies only `package.json`, `node_modules`, and `app.js` from the builder. No dev dependencies or build tools in the final image.
- **Benefits:** Smaller image, faster pulls, reduced attack surface, and production-only runtime.

### Key implementation details

- **`COPY package*.json ./`** — Ensures both `package.json` and `package-lock.json` are present so `npm ci` works and builds are reproducible.
- **Non-root user:** `USER node` so the process does not run as root.
- **Exposed port:** `EXPOSE 3000` for documentation and tooling; actual mapping is done at `docker run`.

---

## 4. Terraform & AWS Design

### Network layout

- **VPC:** `10.0.0.0/16` with DNS hostnames enabled.
- **Public subnet:** `10.0.1.0/24` in a single AZ, with auto-assign public IP.
- **Internet Gateway** and **route table** so instances can reach the internet (e.g. for `docker pull` and package updates).

### Security group

- **Ingress:** SSH (22) and application (3000) from `0.0.0.0/0` for assignment simplicity and direct access from a laptop/browser.
- **Egress:** All traffic allowed so the instance can pull images and install packages.
- **Note:** For production, restrict SSH and app access by IP or use a bastion and/or ALB.

### EC2 instance

- **User data:** `install_docker.sh` installs Docker (official Ubuntu method) and adds `ubuntu` to the `docker` group so that after SSH, Docker can be used without `sudo` if desired (script uses `sudo` for reliability during first boot).
- **Variables:** Region, AMI, instance type, and key pair name are parameterized in `variables.tf` and `terraform.tfvars` so the same code can be reused across environments.

### Output

- **`ec2_public_ip`** — Printed after `terraform apply` so you can SSH and open `http://<ip>:3000` without checking the AWS console.

---

## 5. CI/CD (GitHub Actions) Design

- **Trigger:** Push to `master` so every merge to main produces a new image.
- **Docker Hub push:** The push to Docker Hub happens **when the GitHub Actions workflow runs** (on push to `master`). There is no manual `docker push` required for the pipeline—the workflow builds and pushes in one step.
- **Single job:** Checkout → Docker Buildx → Docker Hub login → build and push. No separate test job in this assignment.
- **Secrets:** `DOCKER_USERNAME` and `DOCKER_PASSWORD` (or token) for Docker Hub; no credentials in the workflow file.
- **Tags:**  
  - `latest` for easy use.  
  - `${{ github.sha }}` for traceability and future rollbacks.
- **Context:** Build context is `./app` so the Dockerfile and app code live together and the repo root stays clean.

---

## 6. Deployment Flow (Manual on EC2)

Terraform does **not** start the container (no Docker run in user data). Deployment on EC2 is done by **cloning the repo, building the image on the instance, and running the container**—not by pulling from Docker Hub.

Steps used:

1. Terraform applies → EC2 is created, Docker installed via `user_data`.
2. SSH to EC2 → clone the repository: `git clone https://github.com/Debjyoti2004/8byte-intern-assignment`
3. Build the Docker image on the instance: `cd 8byte-intern-assignment/app` then `docker build -t 8byte-intern-app .`
4. Run the container: `docker run -p 3000:3000 8byte-intern-app` (or `docker run -d -p 3000:3000 --name 8byte-app 8byte-intern-app` for background).
5. Open `http://<EC2_PUBLIC_IP>:3000` in the browser.

This approach keeps deployment self-contained (clone + build + run) and matches the assignment verification commands. Optional next step: add a small script or systemd unit to run the container on boot.

---

## 7. Challenges & Fixes

Detailed notes are in [CHALLENGES.md](CHALLENGES.md). Summary:

| Issue | Cause | Fix |
|-------|--------|-----|
| `npm ci` failing in Docker | Only `package.json` was copied | Use `COPY package*.json ./` to include `package-lock.json` |
| Terraform `InvalidKeyPair.NotFound` | `key_name` set to local `.pem` path | Set `key_name` to the AWS key pair **name** (e.g. `AWS-key`) |

---

## 8. Documentation & Evidence

- **README.md** — End-to-end instructions: local run, Docker build/run, Terraform init/apply, EC2 deploy, and GitHub Actions. Screenshots in `public/` are linked for Terraform apply, EC2, app in browser, and pipeline success.
- **APPROACH.md** (this file) — Rationale and approach for reviewers.
- **CHALLENGES.md** — Technical issues encountered and how they were resolved.
- **Demo link** — Placeholder in README for a video or walkthrough link.

---

## 9. Summary

The assignment is implemented with a clear split: **app** (Node.js in `app/`), **image** (Dockerfile in `app/`), **infrastructure** (Terraform in `terraform/`), and **automation** (GitHub Actions). Documentation and screenshots are structured so that an evaluator can follow the steps and verify Terraform apply, EC2, app in browser, and a successful pipeline without changing any code.
