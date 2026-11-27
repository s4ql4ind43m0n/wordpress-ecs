# Runbook

Operating `wordpress-cluster` in `wordpress-staging-vpc` (account
`<AWS_ACCOUNT_ID>`, region `us-east-1`).

## Key resource names

| Thing | Value |
|---|---|
| ECS cluster | `wordpress-cluster` |
| WordPress service | `wordpress-service` (task family `wordpress-task`) |
| MySQL service | `mysql-service` (task family `mysql-task`) |
| ALB | `wordpress-alb` — DNS in `aws elbv2 describe-load-balancers` |
| Target group | `wordpress-tg` |
| Log group | `/aws/ecs/wordpress-staging` (streams: `wordpress/nginx`, `wordpress/php`, `wordpress/mysql`) |
| DB secret | `wordpress/staging/database` in Secrets Manager |
| Service discovery | Cloud Map namespace `wordpress-staging`; WordPress reaches MySQL at `mysql` |
| WordPress EFS | `wordpress-efs` (`fs-<WORDPRESS_EFS_ID>`), mounted at `/var/www/html` |
| MySQL EFS | `mysql-efs` (`fs-<MYSQL_EFS_ID>`), mounted at `/var/lib/mysql` |

## Deploying a change

Automated (once GitHub Actions OIDC clears — see testing-results.md):

```bash
git push origin main
```

Manual (what this session actually exercised end-to-end):

```bash
# 1. Build for the platform the task actually runs (ARM64/Graviton)
docker buildx build --platform linux/arm64 -t <account>.dkr.ecr.us-east-1.amazonaws.com/wordpress-php:<tag> --push ./php
docker buildx build --platform linux/arm64 -t <account>.dkr.ecr.us-east-1.amazonaws.com/wordpress-nginx:<tag> --push ./nginx

# 2. Register a new task definition revision (edit ecs/wordpress-task-definition.json's
#    image tags first, or swap them with jq)
aws ecs register-task-definition --cli-input-json file://ecs/wordpress-task-definition.json --region us-east-1

# 3. Point the service at the new revision
aws ecs update-service --cluster wordpress-cluster --service wordpress-service \
  --task-definition wordpress-task:<new-revision> --desired-count 1 --region us-east-1

# 4. Watch it land
aws ecs describe-services --cluster wordpress-cluster --services wordpress-service \
  --region us-east-1 --query 'services[0].deployments'
```

## Rolling back

```bash
aws ecs describe-task-definition --task-definition wordpress-task --region us-east-1  # confirm current revision
aws ecs update-service --cluster wordpress-cluster --service wordpress-service \
  --task-definition wordpress-task:<previous-revision> --desired-count 1 --region us-east-1
```

Verified working this session: deployed revision 3, confirmed serving, rolled
back to revision 2, confirmed healthy again within about a minute.

## Checking health

```bash
ALB_DNS=$(aws elbv2 describe-load-balancers --names wordpress-alb --region us-east-1 --query 'LoadBalancers[0].DNSName' --output text)
curl http://$ALB_DNS/health   # expect: 200, "OK"

aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups --names wordpress-tg --region us-east-1 --query 'TargetGroups[0].TargetGroupArn' --output text) \
  --region us-east-1
```

## Reading logs

```bash
aws logs tail /aws/ecs/wordpress-staging --log-stream-name-prefix wordpress/nginx --follow --region us-east-1
aws logs tail /aws/ecs/wordpress-staging --log-stream-name-prefix wordpress/php --follow --region us-east-1
aws logs tail /aws/ecs/wordpress-staging --log-stream-name-prefix wordpress/mysql --follow --region us-east-1
```

## Replacing a stuck/misbehaving task

The service will replace it automatically on the next health check failure,
but to force it immediately:

```bash
aws ecs list-tasks --cluster wordpress-cluster --service-name wordpress-service --region us-east-1
aws ecs stop-task --cluster wordpress-cluster --task <task-arn> --reason "manual replacement" --region us-east-1
```

Both WordPress files and MySQL data are on EFS, not the task's own disk —
replacing a task does not lose data (verified this session for both).

## If `mysql-service` won't start tasks

This exact failure happened for real this session — every launch attempt
failed with an `AccessDeniedException` on `secretsmanager:GetSecretValue`
from `ecsTaskExecutionRole`. Check:

```bash
aws ecs describe-services --cluster wordpress-cluster --services mysql-service --region us-east-1 --query 'services[0].events[0:5]'
```

If it's a Secrets Manager access error, check `ecsTaskExecutionRole` has a
policy granting `secretsmanager:GetSecretValue` on the specific secret ARN
referenced in the task definition's `secrets` block.

## Rotating the database secret

```bash
aws secretsmanager update-secret --secret-id wordpress/staging/database \
  --secret-string '{"username":"wordpress","password":"<new>","database":"wordpress","root_password":"<new>"}' \
  --region us-east-1
```

Then force a new deployment of both services so the running tasks pick up
the new value (Secrets Manager values are read once, at task start, not
live-reloaded):

```bash
aws ecs update-service --cluster wordpress-cluster --service mysql-service --force-new-deployment --region us-east-1
aws ecs update-service --cluster wordpress-cluster --service wordpress-service --force-new-deployment --region us-east-1
```

## Cost awareness

This account is on a free-tier/credits plan with a limited balance. Resources
that bill continuously regardless of traffic:

- NAT Gateway (`wordpress-staging-nat`) — running since 2026-08-16, dual-AZ mode (2 EIPs)
- ALB (`wordpress-alb`)
- `wordpress-ec2-instance` (t3.micro) — see note below
- Both Fargate services (billed per-second while running; ARM64 is ~20% cheaper than x86_64 for the same size)

**`wordpress-ec2-instance` can likely be stopped now.** It was used this
session to initialize the WordPress EFS filesystem via SSM and isn't part of
the running architecture — WordPress and MySQL both run as ECS tasks, not on
this box. Confirm nothing else depends on it, then:

```bash
aws ec2 stop-instances --instance-ids i-<INSTANCE_ID> --region us-east-1
```

(Stop, not terminate, in case it's needed again for EFS file management —
terminating is a one-way action and wasn't explicitly authorized.)
