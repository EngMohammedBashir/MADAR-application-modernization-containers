# 🧭 START / RESUME HERE — MADAR Phase 05

> 🟡 **LIVE CHECKPOINT — 2026-09-03**  
> This file originally described Day One. Gate 0 and the early implementation gates are now complete, so it now serves as the safest resume point for the next session.

## ✅ Already Completed

```text
🧾 Gate 0 account/cost/artifact preflight       ✅
🔎 Retained AMI recovery                        ✅
🧹 Temporary recovery EC2/EBS/SG cleanup        ✅
🐳 Flask modernization + Docker build           ✅
❤️ Local container validation                   ✅
📦 ECR repository + v1 push                     ✅
🌐 VPC + 4 subnets + IGW + route tables         ✅
🛡️ ALB/ECS/RDS security-group chain             ✅
🐘 Private RDS PostgreSQL                       ✅ available
🔐 Secrets Manager DB connection secret         ✅
🔑 ECS Execution Role trust relationship        ✅
```

## 📍 Resume Exactly Here

Current role:

```text
MADAR-P05-ECS-ExecutionRole
Trust principal: ecs-tasks.amazonaws.com
```

### ▶️ Next action

Attach the standard ECS task execution managed policy, then add narrowly scoped Secrets Manager access for the Phase 05 PostgreSQL secret.

After IAM:

```text
🔑 IAM execution permissions
      ↓
📊 CloudWatch log group
      ↓
🚀 ECS cluster
      ↓
📋 Task definition
      ↓
🏃 Initial Fargate task
      ↓
🐘 Verify task → RDS
      ↓
⚖️ Target group + ALB + ECS service
```

## 🔒 Live Architecture IDs

```text
VPC        vpc-011a441b0a790c458
Public-A   subnet-024a57f44c014ab2a
Public-B   subnet-0726ef657d0ab0ca5
Private-A  subnet-0ba1f1f304eec85cb
Private-B  subnet-0395c2043842856ce
IGW        igw-0df8ed399478fa879
Public RT  rtb-076fdacedac35cd66
Private RT rtb-0ed3daeca13e9987e
ALB-SG     sg-00b9b70e13293ff46
ECS-SG     sg-0d13f6af551e284c8
RDS-SG     sg-00ae439cb916d164b
```

## 🐘 Live Database

```text
DB ID       madar-p05-postgres
DB name     madar_legacy
Engine      PostgreSQL 18.3
Class       db.t4g.micro
Public      false
AZ          us-east-1a
Secret      MADAR/Phase05/Postgres
```

Never print or commit the secret value.

## 📦 Container Artifact

```text
ECR repository  madar-phase05-app
Tag             v1
Digest          sha256:2564714f2668c95ab89c81e95e438a63d14c9d66194ea7eda6a34df59ab99346
App port        8080
Health          /api/health
Readiness       /api/ready
```

## 💰 Cost Reminder

🐘 RDS is currently a live cost clock. Keep the implementation moving and delete it after all required evidence/testing.  
🚫 Do not add NAT Gateway.  
🚫 Do not upgrade the AWS account for this phase.  
⚖️ Create ALB only when the ECS runtime is ready to register targets.

## 📸 Next Meaningful Evidence

Do not screenshot routine IAM commands. The next strong portfolio evidence should prove the ECS task definition/runtime security configuration and then a `RUNNING` Fargate task with logs.

## 🧹 Do Not Forget

Phase 05 is not finished until the failure/scaling tests, cost checkpoint, destructive cleanup and residual-resource audit are complete.
