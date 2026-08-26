# Phase 05 Execution Runbook

> This file is the operational companion to the frozen plan. It is intentionally prepared before deployment and will be updated with exact commands/results as the lab runs.

## Gate 0

Complete `../checklists/00-account-cost-preflight.md` first.

Do not proceed on a `NO-GO` result.

## Gate 1 — Recover the legacy application

Launch a temporary EC2 instance from:

```text
ami-0cbd2e9ec0d6f9168
```

Initial inspection checklist on the recovered host:

```bash
hostnamectl
cat /etc/os-release
python3 --version
ps aux | grep -i -E 'flask|gunicorn|python'
sudo systemctl list-units --type=service | grep -i -E 'madar|flask|gunicorn'
sudo ss -lntp
```

Search likely application locations without copying secrets blindly:

```bash
sudo find /opt /srv /var/www /home -maxdepth 4 \
  -type f \( -name 'app.py' -o -name 'wsgi.py' -o -name 'requirements.txt' -o -name 'pyproject.toml' \) \
  2>/dev/null
```

Inspect systemd service definitions if present:

```bash
sudo systemctl cat <service-name>
```

Record:

- app path,
- startup command,
- Python version,
- port,
- dependencies,
- DB environment/configuration,
- filesystem dependencies,
- log paths,
- hardcoded addresses.

### Secret hygiene

Before copying source, scan filenames/configuration for likely secrets. Never commit values.

```bash
grep -RniE 'password|secret|token|access[_-]?key|private[_-]?key' <app-path> 2>/dev/null
```

Replace real values with placeholders or environment-variable lookups.

### Extract only application artifacts

Target repository content after recovery should look conceptually like:

```text
src/
  app.py
  templates/
  static/
  ...
requirements.txt
```

Do not commit `/etc`, SSH material, `.env`, database credentials, shell histories or machine-specific secrets.

After safe extraction and independent verification, terminate the temporary recovery EC2.

## Gate 2 — Containerize locally

Planned Dockerfile pattern after source assessment:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ .
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "app:app"]
```

The exact Python version/module entrypoint must come from the recovered application, not assumption.

Build/test:

```bash
docker build -t madar-app:p05-local .
docker run --rm -p 8080:8080 \
  --env-file .env.local \
  madar-app:p05-local
```

Validation:

```bash
curl -i http://localhost:8080/api/health
```

Do not push until local container validation passes.

## Gate 3 — ECR

Create one private ECR repository with a Phase 05 name. Capture repository URI.

Use versioned tags such as:

```text
p05-v1
<git-short-sha>
```

Record the final image digest used by ECS.

## Gate 4 — Network / database / IAM foundation

Planned CIDR should avoid conflict with earlier MADAR ranges. Proposed:

```text
VPC                 10.60.0.0/16
Public-A            10.60.1.0/24
Public-B            10.60.2.0/24
Private-DB-A        10.60.11.0/24
Private-DB-B        10.60.12.0/24
```

Do not treat these as final until Gate 0 confirms no conflict.

Security Groups:

```text
ALB-SG
  inbound 80 from 0.0.0.0/0

ECS-SG
  inbound app-port from ALB-SG only

RDS-SG
  inbound 5432 from ECS-SG only
```

Create RDS only after SG/subnet-group foundation exists.

Wait for RDS `available` before initialization.

## Gate 5 — ECS / Fargate

Initial task resources:

```text
CPU     256 (.25 vCPU)
Memory  512 MiB
```

Network:

```text
awsvpc
assignPublicIp = ENABLED
```

Baseline desired count:

```text
1
```

Expected task-definition elements:

- ECR image digest/tag,
- app port 8080 (or recovered application port if intentionally changed),
- `awslogs`,
- Secrets Manager injection,
- execution role,
- minimal/empty task role unless S3 is needed,
- container-level health command if useful.

## Gate 6 — ALB

- target group type `ip`,
- ALB across two public subnets,
- health path `/api/health`,
- HTTP listener 80,
- ECS service registers tasks dynamically.

Capture evidence that healthy targets are task IPs rather than EC2 instances.

## Gate 7 — Validation / failure / scaling

### Functional

```bash
curl -i http://<alb-dns>/api/health
curl -i http://<alb-dns>/api/ready
curl -i http://<alb-dns>/<database-backed-endpoint>
```

### Two-task load balancing

Set desired count to 2 and wait for two healthy targets.

### Task replacement

Stop one running task manually. Record timestamps for:

```text
stopped
→ replacement pending
→ replacement running
→ target initial
→ target healthy
```

### Auto Scaling

Use target tracking on average ECS service CPU. Start with a deliberately observable threshold appropriate for the test.

Generate controlled load with `hey` or `ab`. Capture:

- baseline CPU,
- threshold crossing,
- desired/running count change,
- new target healthy,
- scale-in after load stops.

### RDS dependency failure

Temporarily remove/revoke the ECS-SG → RDS-SG 5432 rule.

Expected:

```text
/api/health remains application-alive
/api/ready or DB-backed API fails predictably
```

Restore the rule and prove recovery.

## Gate 8 — Evidence / cost / cleanup

Before deletion capture:

- architecture IDs,
- ECR image/digest,
- task-definition/service state,
- ALB healthy target evidence,
- RDS private configuration,
- SG chain,
- CloudWatch logs,
- two-target state,
- stopped-task replacement,
- auto-scaling event/metrics,
- dependency failure/recovery,
- Cost Explorer checkpoint.

Then execute `99-cleanup-runbook.md`.
