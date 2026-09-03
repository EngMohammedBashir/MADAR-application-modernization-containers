# 🧹 Phase 05 — Cleanup Runbook

> ⚠️ Execute only after all required functionality, failure/scaling evidence and cost screenshots are captured.  
> 🎯 Goal: return Phase 05 to zero/near-zero ongoing cost while preserving Git evidence and the intentionally retained Phase 03 artifacts.

## 🧠 Principles

- 🧱 remove consumer/service layers before dependencies,
- ⏳ wait for asynchronous AWS deletions,
- 🔍 verify every destructive step,
- 🚫 skip final RDS snapshot unless retention is intentionally approved,
- 🔐 never print secret values,
- 🧾 run a final residual-resource audit before declaring Phase 05 closed.

## 📍 Known Live Foundation at Current Checkpoint

```text
VPC        vpc-011a441b0a790c458
IGW        igw-0df8ed399478fa879
Public RT  rtb-076fdacedac35cd66
Private RT rtb-0ed3daeca13e9987e
ALB-SG     sg-00b9b70e13293ff46
ECS-SG     sg-0d13f6af551e284c8
RDS-SG     sg-00ae439cb916d164b
RDS        madar-p05-postgres
DB subnet  madar-p05-db-subnet-group
Secret     MADAR/Phase05/Postgres
ECR        madar-phase05-app
IAM role   MADAR-P05-ECS-ExecutionRole
```

Additional ECS/ALB/log/scaling resources will be added during later gates and must also be removed.

## 1️⃣ Disable Application Auto Scaling

Remove ECS scalable targets/policies created during the scaling test. Verify no policy continues to manage desired count.

## 2️⃣ Scale ECS Service to Zero

```bash
aws ecs update-service --region us-east-1 --cluster <cluster> --service <service> --desired-count 0
```

Wait until no running service tasks remain.

## 3️⃣ Delete ECS Service

Delete the service only after tasks have drained. Verify service/tasks are gone.

## 4️⃣ Delete ALB Listener / Rules

List and delete the HTTP listener/rules created for the lab.

## 5️⃣ Delete ALB

Delete the Phase 05 ALB and wait until it disappears from `describe-load-balancers`.

## 6️⃣ Delete Target Group

Delete only after no listener/load balancer references it.

## 7️⃣ Deregister Task Definition Revisions

Deregister revisions used by the lab. AWS may retain inactive task-definition metadata; document this accurately rather than claiming physical deletion.

## 8️⃣ Delete ECS Cluster

Only after services/tasks are gone.

## 9️⃣ Delete RDS

Current DB:

```text
madar-p05-postgres
DeletionProtection = false
```

Delete with no final snapshot after evidence is complete:

```bash
aws rds delete-db-instance --region us-east-1 --db-instance-identifier madar-p05-postgres --skip-final-snapshot
```

Wait until the instance disappears.

## 🔟 Delete DB Subnet Group

Delete `madar-p05-db-subnet-group` only after RDS is fully gone.

## 1️⃣1️⃣ Delete Phase 05 Secret

Secret: `MADAR/Phase05/Postgres`.

Use a documented deletion method appropriate for a disposable lab. Never expose the secret value while verifying deletion.

## 1️⃣2️⃣ Delete CloudWatch Log Group

Only after required logs/screenshots have been captured.

## 1️⃣3️⃣ Delete ECR Image / Repository

Default closeout recommendation: remove `madar-phase05-app` because the image is reproducible from source/Dockerfile. If deliberately retained, document the reason and ongoing storage cost.

## 1️⃣4️⃣ Remove IAM Roles / Inline Policies

After ECS no longer references them:

- detach managed execution policies,
- remove Phase 05 secret-access inline policy,
- delete the ECS execution role,
- delete application Task Role if one was created.

## 1️⃣5️⃣ Delete Security Groups

After ALB/ECS/RDS ENIs disappear, delete:

```text
MADAR-P05-ALB-SG
MADAR-P05-ECS-SG
MADAR-P05-RDS-SG
```

Never delete the VPC default security group manually.

## 1️⃣6️⃣ Delete Subnets / Custom Route Tables

Delete all four Phase 05 subnets after dependent ENIs are gone, then delete custom route tables. The VPC main route table disappears with the VPC.

## 1️⃣7️⃣ Detach / Delete IGW

```bash
aws ec2 detach-internet-gateway --region us-east-1 --internet-gateway-id igw-0df8ed399478fa879 --vpc-id vpc-011a441b0a790c458
aws ec2 delete-internet-gateway --region us-east-1 --internet-gateway-id igw-0df8ed399478fa879
```

## 1️⃣8️⃣ Delete Phase 05 VPC

```bash
aws ec2 delete-vpc --region us-east-1 --vpc-id vpc-011a441b0a790c458
```

Verification should return no matching VPC or `InvalidVpcID.NotFound` for direct lookup.

## 🔍 19 — Residual Resource Audit

Check at minimum:

```text
🚀 ECS clusters/services/tasks
⚖️ ALB/listeners/target groups
🐘 RDS instances/subnet groups
🌐 public IPv4 / Elastic IPs
🚫 NAT gateways (should remain none for Phase 05)
🏠 Phase 05 VPC/subnets/routes/IGW
📦 ECR
🔐 Secrets Manager
📊 CloudWatch log groups
🔑 IAM Phase 05 roles/policies
🧩 ENIs if VPC deletion is blocked
```

## 💰 20 — Final Cost Closeout

Capture month-to-date usage/fees, credits and final inventory proof. Account Cost Explorer may include other MADAR phases unless cost-allocation tags provide a narrower view.

## 🧬 21 — Phase 03 Retained Assets

🚫 **Do not automatically delete**:

```text
AMI       ami-0cbd2e9ec0d6f9168
Snapshot  snap-0920a020c47fb6447
S3        madar-operational-files-197821101770
```

After Phase 05 succeeds, make a separate documented decision about whether source/container artifacts have replaced the recovery value of the retained AMI/snapshot.

## 🏁 Cleanup Definition of Done

```text
No running Fargate tasks         ✅ required
No Phase 05 ALB                  ✅ required
No Phase 05 RDS                  ✅ required
No Phase 05 public IPv4          ✅ required
No Phase 05 secret/log surprise  ✅ required
No Phase 05 VPC                  ✅ required
No forgotten IAM role            ✅ required
Final cost checkpoint            ✅ required
Residual audit                   ✅ required
```
