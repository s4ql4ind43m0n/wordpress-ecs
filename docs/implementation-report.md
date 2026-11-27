# Implementation Report

Staging environment: WordPress on ECS Fargate, `wordpress-cluster` /
`wordpress-staging-vpc`, AWS account `<AWS_ACCOUNT_ID>`, region `us-east-1`.
Session date: 2026-08-26.

This was not a build-from-empty. A prior session had already created a
meaningful chunk of the infrastructure. This report distinguishes what
already existed, what was broken, and what was newly created.

## Status matrix

| Requirement | Resource | Status (start of session) | Status (end of session) | Action taken |
|---|---|---|---|---|
| VPC (10.0.0.0/16) | `wordpress-staging-vpc` | COMPLETE | COMPLETE | None — verified CIDR and reused |
| Public/private subnets (2 AZ) | `wordpress-public-a/b`, `wordpress-private-a/b` | COMPLETE | COMPLETE | None |
| Internet Gateway | `igw-<IGW_ID>` | COMPLETE | COMPLETE | None |
| NAT Gateway | `wordpress-staging-nat` (regional/dual-AZ mode) | COMPLETE | COMPLETE | None — see cost note below |
| Route tables | `wordpress-public-rt`, `wordpress-private-rt` | COMPLETE | COMPLETE | None |
| ALB security group | `wordpress-alb-sg` | COMPLETE | COMPLETE | None |
| WordPress/ECS security group | `wordpress-ec2-sg` | PARTIAL | COMPLETE (reused) | Reused as-is; also fronts a pre-existing EC2 instance (see below) |
| MySQL security group | `mysql-sg` | **MISCONFIGURED** | COMPLETE | Added ingress: TCP 3306 from `wordpress-ec2-sg` (was completely empty — blocked everything) |
| EFS security group | `wordpress-efs-sg` | **MISCONFIGURED** | COMPLETE | Added ingress: TCP 2049 from `wordpress-ec2-sg` (previously only allowed `mysql-sg`, which would have blocked WordPress tasks from ever mounting `/var/www/html`) |
| ECR repositories | `wordpress-nginx`, `wordpress-php` | PARTIAL | COMPLETE | Enabled scan-on-push; added a lifecycle policy (expire untagged after 7 days, keep last 10 overall); pushed real SHA-tagged images |
| EFS: WordPress files | `wordpress-efs` (`fs-<WORDPRESS_EFS_ID>`) | COMPLETE but empty | COMPLETE and initialized | Mounted via the existing `wordpress-ec2-instance` over SSM, copied WordPress core (from wordpress.org), wrote `wp-config.php` (reads DB creds from env vars, no secrets embedded), set ownership to uid/gid 33 |
| EFS: MySQL data | `mysql-efs` (`fs-<MYSQL_EFS_ID>`) | COMPLETE | COMPLETE | None — MySQL initializes its own datadir on first boot |
| Secrets Manager | `wordpress/staging/database` | COMPLETE | COMPLETE | Reused as-is (username/password/database/root_password keys, matching spec) |
| IAM: `ecsTaskExecutionRole` | exists | **MISCONFIGURED** | COMPLETE | Added scoped inline policy granting `secretsmanager:GetSecretValue` on the DB secret only — this was the reason `mysql-service` had never once started successfully |
| IAM: `wordpressTaskRole` | exists | COMPLETE | COMPLETE | None — correctly empty (no runtime AWS calls needed yet) |
| IAM: GitHub OIDC provider | none | MISSING | COMPLETE | Created `token.actions.githubusercontent.com` provider, audience `sts.amazonaws.com` |
| IAM: `GitHubActions-ECS-Deploy` role | none | MISSING | COMPLETE (blocked, see testing-results.md) | Created; trust scoped to `repo:s4ql4ind43m0n/wordpress-ecs:environment:staging`; least-privilege policy (ECR push on the 2 repos, ECS register/describe/update scoped to this service, `iam:PassRole` limited to the 2 ECS roles, log write scoped to the 1 log group) |
| ECS cluster | `wordpress-cluster` | COMPLETE | COMPLETE | Reused (differs from the doc's example name `wordpress-staging` — kept the real name rather than creating a duplicate cluster) |
| ECS: `mysql-task` definition | rev 1 | PARTIAL | rev 2 | Added a named port mapping (`mysql`) for Service Connect; moved logging to the shared `/aws/ecs/wordpress-staging` log group |
| ECS: `mysql-service` | ACTIVE, 0/1 running for its entire prior existence | **BROKEN** | COMPLETE, running | Fixed the IAM gap above; fixed network config (was pointing at the **default VPC's** subnets/SG with public IPs enabled — not `wordpress-staging-vpc` at all); enabled ECS Service Connect exposing the service as `mysql` |
| ECS: `wordpress-task` definition | none | MISSING | rev 3 (ARM64) | Created; had to add `runtimePlatform: ARM64` after rev 1 failed to pull (see note) |
| ECS: `wordpress-service` | none | MISSING | COMPLETE, running, healthy | Created in the private subnets, no public IP, attached to `wordpress-tg`, Service Connect client enabled |
| ALB | `wordpress-alb` | COMPLETE | COMPLETE | None — already internet-facing, correct subnets/SG |
| Target group | `wordpress-tg` | COMPLETE (0 targets) | COMPLETE, 1 healthy target | None to the config; now has a real target |
| CloudWatch log group | none (only an unrelated `/ecs/mysql-task`) | MISSING | COMPLETE | Created `/aws/ecs/wordpress-staging`, 14-day retention; all 3 containers streaming to it |
| GitHub repo + staging environment | none | MISSING | COMPLETE | Created `s4ql4ind43m0n/wordpress-ecs`, `staging` environment, all variables + `AWS_ROLE_ARN` secret |
| GitHub Actions workflow | none | MISSING | COMPLETE (blocked, see testing-results.md) | Written, two real bugs found and fixed before it ever ran (see below) |

## Notable finds and fixes

### `mysql-service` had never once run successfully

The service existed, was `ACTIVE`, desired count 1 — and had 0 running tasks
since the day it was created. Every launch attempt failed with:

```
AccessDeniedException: ... ecsTaskExecutionRole/... is not authorized to
perform: secretsmanager:GetSecretValue on resource: ...wordpress/staging/database...
```

`ecsTaskExecutionRole` had only the managed `AmazonECSTaskExecutionRolePolicy`
attached, which does not include Secrets Manager access — the task definition
needs it to fetch `MYSQL_USER`/`MYSQL_PASSWORD`/etc. before the container
starts. Fixed with a narrowly-scoped inline policy (`GetSecretValue` on that
one secret ARN, nothing else).

Separately, the service's network configuration pointed at 6 subnets and a
security group belonging to the **account's default VPC**, not
`wordpress-staging-vpc`, with `assignPublicIp: ENABLED` — the opposite of the
"private subnets, no public IP" requirement. Fixed via
`aws ecs update-service --network-configuration ...` to use the correct
private subnets and `mysql-sg`, with public IPs disabled.

### WordPress images built on Apple Silicon wouldn't pull on Fargate

The Nginx and PHP-FPM images were built locally on an arm64 Mac, producing
arm64-only manifests. Fargate defaults to x86_64, so the first `wordpress-task`
revision failed every launch with `CannotPullContainerError: ... platform
'linux/amd64'`. Rather than rebuild for amd64, the task definition now sets
`runtimePlatform: { cpuArchitecture: ARM64 }` — Fargate ARM64 (Graviton) is
also roughly 20% cheaper per vCPU/GB, which matters on this account's limited
credit balance. The GitHub Actions workflow was updated to cross-build for
`linux/arm64` via Buildx/QEMU so future CI-driven deploys keep matching.

### A workflow bug caught before it ever ran

The initial `deploy.yml` downloaded the "current task definition" by
`$ECS_SERVICE` (a service name) instead of the task definition family name.
Caught by re-reading the file before the first real run; fixed by adding a
dedicated `ECS_TASK_FAMILY` variable.

### The existing EC2 instance turned out to be exactly what it looked like

`wordpress-ec2-instance` (t3.micro, running since 2026-08-16, in the private
subnet, SSM-managed) had no WordPress files, no NFS tooling installed, and no
purpose fulfilled yet — but its name, its VPC, its key pair name
(`wordpress-ec2-key`, matching a `.pem` file already in the account owner's
Downloads folder), and the existence of a dedicated `wordpress-ssm-role`
strongly suggested it was provisioned specifically to initialize the EFS
filesystems before any ECS task existed to do it. It was used for exactly
that (see "EFS: WordPress files" row above) via SSM `send-command` — no SSH,
no credentials handled directly.

## What's still open

- **GitHub Actions OIDC deploy is configured but not yet verified end-to-end** —
  blocked by what looks like an AWS new-account verification hold, not a
  configuration problem. See `docs/testing-results.md` test 10 for the full
  diagnosis. Nothing further to do here except re-run
  `gh workflow run deploy.yml -R s4ql4ind43m0n/wordpress-ecs` once the account
  finishes verifying.
- Container Insights is not enabled on `wordpress-cluster` (left off deliberately
  given the account's limited remaining credit — it adds CloudWatch custom-metric
  cost. Trivial to turn on: `aws ecs update-cluster-settings --cluster
  wordpress-cluster --settings name=containerInsights,value=enhanced`).
- ~~Autoscaling~~ — added 2026-08-26: `wordpress-service` now has an
  Application Auto Scaling target (min 1, max 3, target-tracking on 70% CPU).
- HTTPS/ACM and WAF are explicitly deferred by decision (no domain available;
  WAF's ongoing cost not justified yet on this account's remaining credit),
  not oversights — see `docs/production-gap-analysis.md`. The Aurora/RDS
  migration remains a staging-to-production gap, also covered there.

## Evidence

No screenshot image files are committed to this repo — the browser
automation available in this session has no mechanism to save a screenshot
to disk (only display it for inline verification), and a direct macOS
`screencapture` attempt confirmed there's no attached display session to
capture from either. This is a tool limitation, verified rather than assumed.

What exists instead: a full visual walkthrough of the live AWS console was
done at the end of the build (VPC, subnets, the `mysql-sg` fix, ECR with a
real pushed image, both EFS filesystems, the `ecsTaskExecutionRole` IAM fix,
the ECS cluster showing 2 active services, the `wordpress-service` health
tab with its ALB target showing 1 healthy, the ALB itself, the NAT Gateway,
and CloudWatch log streams with real timestamps) plus the live WordPress
site — all confirmed against the actual running account, described in full
in this session's record. Anyone with console access to this AWS account can
reproduce every one of those views from the URLs and resource names in
`docs/step-by-step-guide.md`. Command output (JSON from `aws` CLI calls, HTTP
status codes, CloudWatch log excerpts) remains the primary durable evidence
trail in `docs/testing-results.md`, since it doesn't depend on anyone
re-opening a console.
