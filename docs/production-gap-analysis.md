# Production Gap Analysis

What's built is a correct **staging** environment. This lists what would need
to change before calling it production-grade, and why each gap matters.

## Database: MySQL-on-ECS/EFS vs. Aurora/RDS

**Current:** MySQL runs as its own ECS Fargate service, storing data on an
EFS filesystem (`mysql-efs`), reached from WordPress via ECS Service Connect
(`DB_HOST=mysql`).

**Why that's fine for staging:** it's cheap, it's fully within the ECS/Fargate
mental model, and this session verified data survives a task replacement.

**Why it's not the production answer:**
- EFS is a general-purpose network filesystem, not a database storage engine.
  It works, but it isn't tuned for InnoDB's I/O patterns the way EBS
  (RDS/Aurora's backing store) is — expect worse and less predictable latency
  under real load.
- No automated backups, point-in-time recovery, read replicas, or Multi-AZ
  failover. A bad `DROP TABLE` or a corrupted task means restoring from
  whatever you remembered to snapshot yourself.
- Single point of failure: one MySQL task. If it crashes mid-write, you're
  relying on InnoDB's crash recovery on a filesystem it wasn't designed for.

**Production fix:** WordPress ECS -> Aurora MySQL (or RDS MySQL if Aurora is
overkill for the traffic level). Multi-AZ, automated backups, and read
replicas become configuration, not custom-built ECS services.

## HTTPS / ACM

**Current:** ALB listens on HTTP:80 only.

**Gap:** all traffic, including the WordPress admin login, travels
unencrypted. Request an ACM certificate for the real domain, add an
HTTPS:443 listener forwarding to the same target group, and redirect
HTTP -> HTTPS on the existing :80 listener.

## Route 53

**Current:** the ALB is reached by its raw `*.elb.amazonaws.com` DNS name.

**Gap:** no custom domain, so ACM's DNS validation, any future CDN in front,
and basic professionalism all need a real hostname. Add a Route 53 hosted
zone (or point an existing one) with an ALIAS record to the ALB.

## WAF

**Current:** nothing in front of the ALB filtering malicious requests.

**Gap:** WordPress is one of the most commonly-scanned platforms on the
internet (`wp-login.php`, XML-RPC abuse, plugin-vulnerability probes). AWS
WAF with the managed WordPress rule group, plus basic rate-limiting, is cheap
insurance relative to the blast radius of a compromised admin account.

## Autoscaling

**Current:** both services are fixed at desired count 1.

**Gap:** no response to load, and no resilience if the single task's AZ has
an issue. Add an ECS Service Auto Scaling target tracking CPU/memory (or ALB
request count per target) for `wordpress-service`, minimum 2 tasks spread
across the 2 AZs. `mysql-service` should generally stay at 1 (it's not
horizontally scalable in this design) — that's exactly why it shouldn't be
MySQL-on-ECS in production either.

## Multi-AZ resilience

**Current:** desired count 1 for both services means, in practice, "runs in
whichever AZ ECS happens to place it."

**Gap:** this is really the same gap as autoscaling — desired count 1 means
zero redundancy by definition. Fixed by the autoscaling change above for
WordPress; fixed by the Aurora Multi-AZ change for the database.

## Backup strategy / disaster recovery

**Current:** whatever EFS's default durability gives you (EFS itself
replicates data across AZs, so filesystem loss isn't the main risk) — but
there is no point-in-time restore for either the database or WordPress
content, and no tested recovery procedure.

**Gap:** for the database, this is subsumed by the Aurora/RDS migration
(automated snapshots, PITR). For WordPress files (mainly `wp-content/`), add
scheduled EFS-to-S3 backups via AWS Backup, and actually test a restore once.

## Least-privilege IAM

**Current:** reasonably tight already —
`ecsTaskExecutionRole` only has the one added secret ARN beyond the AWS
managed policy; `GitHubActions-ECS-Deploy` is scoped to the specific ECR
repos, the specific ECS service, and `iam:PassRole` is limited to exactly the
two ECS roles that need it.

**Gap:** `ecs:RegisterTaskDefinition` / `DescribeTaskDefinition` had to stay
`Resource: "*"` — ECS doesn't support resource-level restriction on those
actions. That's a real, unavoidable AWS limitation, not an oversight; worth
knowing rather than assuming it's tightenable later.

## ECR vulnerability scanning

**Current:** scan-on-push is now enabled for both repos (this session), with
a lifecycle policy retaining the last 10 images and expiring untagged images
after 7 days.

**Gap:** nothing currently *acts* on scan findings — no pipeline gate that
blocks a deploy if a HIGH/CRITICAL CVE shows up. Worth adding a step in
`deploy.yml` that checks `aws ecr describe-image-scan-findings` before the
deploy step proceeds.

## Deployment circuit breaker

**Current:** the ECS deployment configuration observed on the running
services already has `deploymentCircuitBreaker: { enable: true, rollback:
true }` (this appears to be an ECS default in this account/region, not
something explicitly configured this session).

**Gap:** none functionally, but it's worth deliberately setting it in the
task definitions / `create-service` calls going forward rather than relying
on defaults that could differ elsewhere.

## Monitoring / alarms

**Current:** logs flow to CloudWatch (`/aws/ecs/wordpress-staging`); no
alarms exist.

**Gap:** no one gets paged if the ALB starts returning 5xx, if CPU pins, or
if the MySQL task stops. Minimum viable set: ALB `HTTPCode_Target_5XX_Count`,
ALB `UnHealthyHostCount`, ECS service CPU/memory utilization, and an alarm on
`mysql-service` running count dropping below 1.

## Private networking / VPC endpoints

**Current:** private-subnet tasks reach ECR, Secrets Manager, and CloudWatch
Logs through the NAT Gateway (which is already running and already billed
regardless).

**Gap:** for a genuinely locked-down design, add VPC endpoints for ECR (both
`ecr.api` and `ecr.dkr`), Secrets Manager, and CloudWatch Logs, and tighten
the NAT/security-group egress rules once those endpoints exist. This mainly
matters for reducing NAT data-processing cost at scale and for defense in
depth — it is not blocking anything today.

## WordPress media persistence strategy

**Current:** `wp-content/uploads/` lives on the same EFS filesystem as the
rest of WordPress core.

**Gap:** works fine at this scale. At real scale, move uploads to S3 (via a
plugin like WP Offload Media, or a custom stream wrapper) so media serving
doesn't compete with EFS throughput used by PHP execution, and so CDN
(CloudFront) integration is straightforward.

## Container security

**Current:** containers run as their image's default user (root for the PHP
image's PHP-FPM master process, though workers drop to `www-data`; `nginx`'s
worker processes run as `nginx`/uid 101 in the official image).

**Gap:** no explicit `readonlyRootFilesystem`, no dropped Linux capabilities,
no non-root enforcement at the task-definition level. Worth hardening once
the environment is otherwise stable — these are task-definition-level
settings (`linuxParameters`), not image rebuilds.

## Rollback strategy

**Current:** verified working this session (deployed `wordpress-task:3`,
rolled back to `:2`, confirmed healthy) — this is already production-adequate
as a mechanism. The gap is process, not tooling: there's no documented
"who decides to roll back and how fast" runbook step beyond the command
itself (now added to `docs/runbook.md`).
