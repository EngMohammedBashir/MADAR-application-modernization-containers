# 🧰 Phase 05 — Rebuild / Execution Runbook

> 🟢 **Historical execution: COMPLETE.** This runbook is written so I—or another engineer—can understand and rebuild the lab without needing the original chat.  
> 🔐 Never print or commit passwords, secret values, access keys or private keys.

## 🧠 Mental Model

```text
Legacy VM artifact → recover source → container → registry → Fargate → ALB
                                      ↓                     ↓
                                external config          private RDS
```

## 0️⃣ Preflight

Verify identity/region and retained assets before spending money.

```powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-images --image-ids ami-0cbd2e9ec0d6f9168 --region us-east-1
aws ec2 describe-snapshots --snapshot-ids snap-0920a020c47fb6447 --region us-east-1
aws s3api head-bucket --bucket madar-operational-files-197821101770
```

**Why:** prove I am operating in the intended account/region and that continuity artifacts exist. **Expected:** AMI `available`, snapshot `completed`, S3 accessible.

## 1️⃣ Recover the Application

Launch a temporary recovery EC2 from the retained AMI only long enough to inspect `/home/madaradmin/madar-legacy-app` and copy the required source. Imported VM images may preserve guest SSH configuration instead of behaving like a native cloud image; do not assume a newly selected EC2 key will work.

I recovered `app.py`, templates, static assets and required scripts, then deleted the temporary EC2/EBS/SG.

## 2️⃣ Modernize the Runtime

The application must not depend on `localhost` PostgreSQL or embedded credentials. Use environment variables:

```text
MADAR_DB_HOST
MADAR_DB_PORT
MADAR_DB_NAME
MADAR_DB_USER
MADAR_DB_PASSWORD
```

Container standard:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && useradd --create-home --shell /usr/sbin/nologin madar
COPY app.py .
COPY templates ./templates
COPY static ./static
RUN chown -R madar:madar /app
USER madar
EXPOSE 8080
CMD ["gunicorn","--bind","0.0.0.0:8080","--workers","2","--access-logfile","-","--error-logfile","-","app:app"]
```

Build/test:

```powershell
docker build -t madar-phase05-app:local .
docker run -d --name madar-app -p 8080:8080 -e MADAR_DB_PASSWORD=local-test-only madar-phase05-app:local
curl.exe -i http://localhost:8080/api/health
curl.exe -i http://localhost:8080/api/ready
docker exec madar-app id
```

**Expected:** health `200`; readiness can fail locally because no PostgreSQL is present; `id` must show non-root `madar`.

## 3️⃣ Publish Application Image

```powershell
aws ecr create-repository --repository-name madar-phase05-app --image-scanning-configuration scanOnPush=true --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 197821101770.dkr.ecr.us-east-1.amazonaws.com
docker tag madar-phase05-app:local 197821101770.dkr.ecr.us-east-1.amazonaws.com/madar-phase05-app:v1
docker push 197821101770.dkr.ecr.us-east-1.amazonaws.com/madar-phase05-app:v1
```

**Why:** Fargate pulls an immutable deployment artifact from ECR rather than from my workstation.

## 4️⃣ Build Network + Security

Create dedicated VPC `10.60.0.0/16`, two public and two private subnets across `us-east-1a/b`, IGW/public routing and local-only private routing. No NAT was used in this lab.

Security rule chain:

```text
Internet 0.0.0.0/0 → ALB-SG TCP/80
ALB-SG              → ECS-SG TCP/8080
ECS-SG              → RDS-SG TCP/5432
```

**Rule:** do not add `0.0.0.0/0` to ECS `8080` or RDS `5432`.

## 5️⃣ Create RDS + Secret

Create private PostgreSQL RDS in the two private subnets. Lab configuration was `db.t4g.micro`, 20 GiB gp3, Single-AZ, `PubliclyAccessible=false`.

Store DB connection JSON in Secrets Manager. Verify **keys**, never the secret value:

```text
username / password / host / port / dbname
```

⚠️ **Actual error:** my first Fargate task failed because the secret JSON was malformed. I corrected the JSON structure, verified key names safely, and the next task started.

## 6️⃣ Recover and Restore Legacy DB

S3 did not contain the full dump. I mounted a temporary EBS volume created from retained snapshot `snap-0920a020c47fb6447` read-only and found the authoritative custom dump. I copied it to:

```text
s3://madar-operational-files-197821101770/database-backups/madar_legacy_final.dump
```

A dedicated restore task role had only:

```text
s3:GetObject → arn:aws:s3:::madar-operational-files-197821101770/database-backups/madar_legacy_final.dump
```

The restore container used `aws s3 cp` + `pg_restore --no-owner --no-acl --exit-on-error`. Restore task exited `0` and logged `RESTORE COMPLETED SUCCESSFULLY`.

## 7️⃣ ECS Task + ALB + Service

Register Fargate task definition with `awsvpc`, CPU `256`, memory `512`, image `:v1`, container port `8080`, CloudWatch logs and secret injection. Place tasks in public subnets with public IPv4 **only because this lab intentionally avoided NAT**; ECS-SG still accepts app ingress only from ALB-SG.

Create target group `ip:8080`, health path `/api/health`, Internet-facing ALB on public A/B, listener `HTTP:80`, then ECS service attached to TG.

Validate:

```powershell
curl.exe -i http://<ALB-DNS>/api/health
curl.exe -i http://<ALB-DNS>/api/ready
```

**Expected:** both `200` with DB connected on readiness.

## 8️⃣ Self-Healing Test

Set desired count to `2`, wait for two healthy targets, intentionally stop one service task, then observe ECS replace it while ALB drains the old target.

**Proof required:** desired/running mismatch during failure + replacement task + two healthy targets after recovery.

## 9️⃣ Target-Tracking Scale-Out

Register ECS scalable target `Min=1 Max=2` and CPU target tracking. Intended target was `40%`. For controlled proof I temporarily used `5%`, generated load, observed CloudWatch ALARM and automatic desired count `1→2`, then restored `40%`.

⚠️ Do not claim scale-in unless separately evidenced.

## 🔟 DB Dependency Failure

Temporarily revoke only the ECS-SG→RDS-SG TCP/5432 ingress rule. Observe through ALB, then immediately restore the exact rule.

Observed:

```text
failure  health=200 / ready=502
recovery health=200 / ready=200
```

## 1️⃣1️⃣ Observability

Capture ECS desired/running/pending, target health, RDS status/private flag, log streams, ECS CPU/memory, ALB requests and RDS connections. Screenshots should prove a claim, not just show a console page.

## 1️⃣2️⃣ Cost + Cleanup

Capture Cost Explorer checkpoint, then execute `99-cleanup-runbook.md`. Do not leave ALB/RDS/Fargate/public IPv4 resources running after evidence collection.

## 🧯 Troubleshooting Lessons

- `TaskFailedToStart` + secret injection → validate secret JSON keys/ARN suffix; never print password.
- Public Fargate IP times out directly → check SG source; in this lab that timeout was expected security behavior.
- Missing DB backup → inspect retained artifacts rather than fabricating data.
- Scaling not triggering → inspect actual CPU datapoints/alarm evaluation before changing policy; restore intended target after controlled test.
- AWS waiter fails with `ExpiredToken` → refresh credentials and rerun verification. Do not print a success message unconditionally after a failed command.

## 🏁 Independent Rebuild Rule

At every gate use: **create → verify → capture only meaningful evidence → continue**. Never trust a command because it returned no text; verify the resulting AWS state.