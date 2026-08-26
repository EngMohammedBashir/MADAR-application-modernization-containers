# Phase 05 Cleanup Runbook

> Execute only after all required evidence and cost snapshots are captured.

## Principles

- remove resources from consumer/service layer downward,
- wait for asynchronous deletions before removing dependencies,
- verify every destructive step,
- do not create a final RDS snapshot unless retention is intentionally approved,
- run a residual-resource audit before declaring Phase 05 closed.

## 1. Disable application auto scaling

Before deleting the ECS service, remove scalable targets/policies created for the service.

Verify no scaling policy continues to manage desired count.

## 2. Scale ECS service to zero

```bash
AWS_PAGER="" aws ecs update-service \
  --region us-east-1 \
  --cluster <cluster> \
  --service <service> \
  --desired-count 0
```

Wait until no running tasks remain.

## 3. Delete ECS service

```bash
AWS_PAGER="" aws ecs delete-service \
  --region us-east-1 \
  --cluster <cluster> \
  --service <service>
```

Verify service/tasks are gone before continuing.

## 4. Delete ALB listener/rules

List listeners first:

```bash
AWS_PAGER="" aws elbv2 describe-listeners \
  --region us-east-1 \
  --load-balancer-arn <alb-arn> \
  --output table
```

Delete the listener(s), then verify none remain.

## 5. Delete ALB

```bash
AWS_PAGER="" aws elbv2 delete-load-balancer \
  --region us-east-1 \
  --load-balancer-arn <alb-arn>
```

Wait until `describe-load-balancers` no longer returns the ALB.

## 6. Delete target group

After the load balancer/listener no longer references it:

```bash
AWS_PAGER="" aws elbv2 delete-target-group \
  --region us-east-1 \
  --target-group-arn <target-group-arn>
```

## 7. Deregister ECS task-definition revisions

List the Phase 05 task definitions, then deregister revisions used by the lab.

Historical `INACTIVE` task definition metadata may remain visible; document this accurately rather than claiming physical deletion if AWS retains metadata.

## 8. Delete ECS cluster

Only after services/tasks are gone.

## 9. Delete RDS

Confirm:

```text
DeletionProtection = false
```

Then delete with no final snapshot:

```bash
AWS_PAGER="" aws rds delete-db-instance \
  --region us-east-1 \
  --db-instance-identifier <db-id> \
  --skip-final-snapshot
```

Wait until the DB instance disappears from inventory.

## 10. Delete DB subnet group

Delete only after the RDS instance is fully removed.

## 11. Delete temporary Secrets Manager secret

Choose one documented method:

- schedule deletion with recovery window, or
- force immediate deletion for a disposable lab secret when intentionally approved.

Do not print secret values during verification.

## 12. Delete CloudWatch log group

Only after screenshots/log evidence needed for the portfolio has been captured.

## 13. Delete ECR images/repository

The default closeout recommendation is to remove the container image and repository because the image is reproducible from Git-tracked source/Dockerfile.

If deliberately retained, record the ongoing storage reason/cost rather than silently leaving it.

## 14. Delete Phase 05 security groups

Order usually becomes possible after ALB/ECS ENIs/RDS ENIs are gone.

Delete custom groups only; never delete the VPC default security group manually.

## 15. Delete subnets / route tables

Delete private/public subnets after dependent ENIs/resources disappear.

Delete custom route tables after subnet associations are removed.

The VPC main route table remains and disappears with the VPC.

## 16. Detach/delete Internet Gateway

```bash
AWS_PAGER="" aws ec2 detach-internet-gateway \
  --region us-east-1 \
  --internet-gateway-id <igw-id> \
  --vpc-id <vpc-id>

AWS_PAGER="" aws ec2 delete-internet-gateway \
  --region us-east-1 \
  --internet-gateway-id <igw-id>
```

## 17. Delete Phase 05 VPC

```bash
AWS_PAGER="" aws ec2 delete-vpc \
  --region us-east-1 \
  --vpc-id <vpc-id>
```

Post-delete verification should return no matching VPC or `InvalidVpcID.NotFound` for direct ID lookup.

## 18. Residual-resource audit

Run targeted checks for Phase 05 naming/tag prefix:

```bash
AWS_PAGER="" aws ecs list-clusters --region us-east-1 --output table
AWS_PAGER="" aws elbv2 describe-load-balancers --region us-east-1 --output table
AWS_PAGER="" aws rds describe-db-instances --region us-east-1 --output table
AWS_PAGER="" aws ec2 describe-addresses --region us-east-1 --output table
AWS_PAGER="" aws ec2 describe-nat-gateways --region us-east-1 --output table
AWS_PAGER="" aws ec2 describe-vpcs --region us-east-1 --output table
AWS_PAGER="" aws ecr describe-repositories --region us-east-1 --output table
```

Also check:

- Secrets Manager,
- CloudWatch log groups,
- ECS task-definition history,
- ENIs if a VPC deletion is blocked.

## 19. Final cost closeout

Capture:

- month-to-date Usage/Fee,
- credits,
- calculated/account net view,
- final inventory proof.

State clearly that account-level Cost Explorer values may include other MADAR phases unless cost-allocation tags provide a narrower view.

## 20. Retained Phase 03 artifacts

Do **not** automatically delete the Phase 03 AMI/snapshot/S3 as part of this runbook.

After Phase 05 succeeds, make a separate documented decision about whether container/source artifacts have replaced the recovery value of:

```text
ami-0cbd2e9ec0d6f9168
snap-0920a020c47fb6447
madar-operational-files-197821101770
```
