# Step-by-Step Implementation Guide

This walks through all 30 steps of the original design in build order. Every
command here was actually run against a real AWS account during this
project's build — this isn't a plan, it's a record of what worked, with the
bugs that came up along the way and how they were fixed. Replace
`536055837898` / `us-east-1` / `s4ql4ind43m0n/wordpress-ecs` with your own
account ID, region, and repo.

Each step says **what** to do, **why it matters**, and **how to know it
worked**.

---

## Phase A — Networking (steps 1-4)

### 1. VPC

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region us-east-1 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=wordpress-staging-vpc}]'
```

**Why it matters:** every other resource — subnets, security groups, the
ALB, ECS tasks — lives inside this network boundary. Get the CIDR wrong here
and you're re-doing every subnet after it.

**Verify:** `aws ec2 describe-vpcs --filters Name=tag:Name,Values=wordpress-staging-vpc`

### 2. Subnets

```bash
VPC_ID=vpc-xxxxxxxx
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=wordpress-public-a}]'
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=wordpress-public-b}]'
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=wordpress-private-a}]'
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=wordpress-private-b}]'
```

**Why it matters:** the ALB goes in the public subnets; ECS tasks and EFS
mount targets go in the private ones and never get a public IP. This is the
whole security model in two sentences — get this split right and most of the
security-group work becomes "allow the ALB to reach ECS," not "figure out
how to hide things from the internet."

**Verify:** 4 subnets, 2 AZs, `MapPublicIpOnLaunch` doesn't actually matter
for the ALB (it gets public IPs regardless as an internet-facing ALB) or the
NAT Gateway (its EIP is explicit) — don't burn time on it.

### 3. Internet Gateway + NAT Gateway

```bash
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=wordpress-igw}]'
aws ec2 attach-internet-gateway --internet-gateway-id igw-xxxx --vpc-id $VPC_ID

aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id subnet-<public-a> --allocation-id eipalloc-xxxx \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=wordpress-staging-nat}]'
```

Then two route tables:

```bash
# Public: 0.0.0.0/0 -> IGW
aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=wordpress-public-rt}]'
aws ec2 create-route --route-table-id rtb-public --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxxx
aws ec2 associate-route-table --route-table-id rtb-public --subnet-id subnet-<public-a>
aws ec2 associate-route-table --route-table-id rtb-public --subnet-id subnet-<public-b>

# Private: 0.0.0.0/0 -> NAT
aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=wordpress-private-rt}]'
aws ec2 create-route --route-table-id rtb-private --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxxx
aws ec2 associate-route-table --route-table-id rtb-private --subnet-id subnet-<private-a>
aws ec2 associate-route-table --route-table-id rtb-private --subnet-id subnet-<private-b>
```

**Why it matters:** private-subnet ECS tasks still need outbound internet
access — to pull images from ECR, reach Secrets Manager, send logs to
CloudWatch. NAT Gateway is what gives them that without a public IP. This is
also the single biggest *unavoidable* recurring cost in the whole stack
(~$32-65/month depending on mode) — there's no staging-cheap alternative to
a working NAT path short of adding VPC endpoints for every AWS service you
call (see `production-gap-analysis.md`).

**Verify:** `aws ec2 describe-route-tables` — private RT's `0.0.0.0/0` route
should target the NAT Gateway, public RT's should target the IGW. Get this
backwards and ECS tasks in private subnets can't pull their own images.

### 4. Security groups

```bash
ALB_SG=$(aws ec2 create-security-group --group-name wordpress-alb-sg --description "ALB" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0

ECS_SG=$(aws ec2 create-security-group --group-name wordpress-ecs-sg --description "WordPress ECS tasks" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ECS_SG --protocol tcp --port 80 --source-group $ALB_SG

MYSQL_SG=$(aws ec2 create-security-group --group-name mysql-sg --description "MySQL ECS Fargate service" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $MYSQL_SG --protocol tcp --port 3306 --source-group $ECS_SG

EFS_SG=$(aws ec2 create-security-group --group-name wordpress-efs-sg --description "EFS mount targets" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $EFS_SG --protocol tcp --port 2049 --source-group $ECS_SG
aws ec2 authorize-security-group-ingress --group-id $EFS_SG --protocol tcp --port 2049 --source-group $MYSQL_SG
```

**Why it matters — this is where it's easy to get burned.** Every rule here
is `security-group -> security-group`, never a raw CIDR except the ALB's
public-facing 80/443. Two real bugs found in this exact spot during this
build:

- `mysql-sg` had been created with **zero** inbound rules — not "too open,"
  completely closed, including to the app that needed it. The service had
  never once started a task because of it.
- `wordpress-efs-sg` only allowed port 2049 from `mysql-sg`, not from the
  WordPress/ECS security group — meaning WordPress's own EFS mount for
  `/var/www/html` would have hung forever waiting on a connection nothing
  was allowed to make.

If both filesystems (WordPress files, MySQL data) share one EFS security
group, that group needs an inbound 2049 rule from **every** security group
that mounts either filesystem — easy to add one and forget the other.

**Verify:** describe each SG's inbound rules and check the source is a
security group ID, not `0.0.0.0/0` (except the ALB's 80/443). Never expose
3306 to the internet.

---

## Phase B — Storage & Secrets (steps 5-8)

### 5. ECR

```bash
aws ecr create-repository --repository-name wordpress-nginx --region us-east-1
aws ecr create-repository --repository-name wordpress-php --region us-east-1

for repo in wordpress-nginx wordpress-php; do
  aws ecr put-image-scanning-configuration --repository-name $repo --image-scanning-configuration scanOnPush=true
  aws ecr put-lifecycle-policy --repository-name $repo --lifecycle-policy-text '{
    "rules": [
      {"rulePriority":1,"description":"Expire untagged after 7 days","selection":{"tagStatus":"untagged","countType":"sinceImagePushed","countUnit":"days","countNumber":7},"action":{"type":"expire"}},
      {"rulePriority":2,"description":"Keep last 10 images","selection":{"tagStatus":"any","countType":"imageCountMoreThan","countNumber":10},"action":{"type":"expire"}}
    ]
  }'
done
```

**Watch out:** if your lifecycle policy uses `"tagStatus": "tagged"`, it
*must* include a `tagPrefixList` (or `tagPatternList`) that actually matches
your real tags. A policy written for `v1.2.3`-style tags silently does
nothing if you're tagging bare git SHAs like `a13f4c2` — this happened
during this build; using `"tagStatus": "any"` for the retention rule sidesteps
it entirely.

**Why it matters:** scan-on-push catches known CVEs in your base images
before they ever run. The lifecycle policy is what stops ECR storage cost
from growing forever as CI pushes a new image on every commit.

### 6. EFS

```bash
WP_FS=$(aws efs create-file-system --encrypted --tags Key=Name,Value=wordpress-efs --query FileSystemId --output text)
DB_FS=$(aws efs create-file-system --encrypted --tags Key=Name,Value=mysql-efs --query FileSystemId --output text)

for fs in $WP_FS $DB_FS; do
  aws efs create-mount-target --file-system-id $fs --subnet-id subnet-<private-a> --security-groups $EFS_SG
  aws efs create-mount-target --file-system-id $fs --subnet-id subnet-<private-b> --security-groups $EFS_SG
done
```

**Why it matters:** this is what makes WordPress files and MySQL data
survive an ECS task being replaced (a deploy, a crash, a host issue) — Fargate
tasks have no durable local disk otherwise. Both persistence tests in
`testing-results.md` (stop the task, confirm data survives) exist specifically
to prove this actually works, not just that it's configured.

### 7. Secrets Manager

```bash
aws secretsmanager create-secret --name wordpress/staging/database --secret-string '{
  "username": "wordpress",
  "password": "<strong-random-password>",
  "database": "wordpress",
  "root_password": "<strong-random-root-password>"
}'
```

**Why it matters:** this is the one and only place the database password
lives. Not in the task definition, not in GitHub Secrets, not in an
environment variable baked into an image. ECS fetches it at task-launch time
using the execution role (next step) and injects it as a container
environment variable — the value never touches your CI pipeline or your
git history.

### 8. IAM roles

```bash
# Execution role - lets ECS pull images, write logs, and fetch secrets
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document '{
  "Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]
}'
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# The managed policy does NOT include Secrets Manager access - add it explicitly, scoped to the one secret:
SECRET_ARN=$(aws secretsmanager describe-secret --secret-id wordpress/staging/database --query ARN --output text)
aws iam put-role-policy --role-name ecsTaskExecutionRole --policy-name AllowReadWordpressDatabaseSecret --policy-document "{
  \"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"secretsmanager:GetSecretValue\",\"Resource\":\"$SECRET_ARN\"}]
}"

# Task role - what the application itself can call at runtime. Empty for now (WordPress doesn't call AWS APIs).
aws iam create-role --role-name wordpressTaskRole --assume-role-policy-document '{
  "Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]
}'
```

**Why it matters — this exact gap took down the MySQL service for its
entire lifetime in this build.** `AmazonECSTaskExecutionRolePolicy` covers
ECR pulls and CloudWatch Logs, but **not** Secrets Manager. Skip the
`put-role-policy` step above and every task launch fails with:

```
AccessDeniedException: ... is not authorized to perform:
secretsmanager:GetSecretValue on resource: ...
```

— with the service silently retrying forever at 0 running tasks, no loud
failure, just an ACTIVE service that never does anything. Check
`aws ecs describe-services ... --query 'services[0].events[0:5]'` if a
service won't start; this exact error will be right there.

Keep the execution role and task role separate — don't grant the execution
role application permissions "since it's already there." The execution
role's job is getting the container started; the task role's job is what
the running application can do.

---

## Phase C — Compute foundation (steps 9-10)

### 9. ECS cluster

```bash
aws ecs create-cluster --cluster-name wordpress-cluster
aws ecs update-cluster-settings --cluster wordpress-cluster --settings name=containerInsights,value=enhanced  # optional, has CloudWatch cost
```

### 10. ALB + target group

```bash
ALB_ARN=$(aws elbv2 create-load-balancer --name wordpress-alb --type application --scheme internet-facing \
  --subnets subnet-<public-a> subnet-<public-b> --security-groups $ALB_SG --query 'LoadBalancers[0].LoadBalancerArn' --output text)

TG_ARN=$(aws elbv2 create-target-group --name wordpress-tg --protocol HTTP --port 80 --vpc-id $VPC_ID \
  --target-type ip --health-check-path /health --health-check-port traffic-port \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

**Why it matters:** `--target-type ip` (not `instance`) is what makes this
work with Fargate at all — Fargate tasks aren't EC2 instances, so the target
group has to track them by ENI IP address instead. The health check path
must match whatever your Nginx config actually serves at `/health` (this
repo's `nginx/default.conf` returns a bare `200 OK` there, on purpose, so it
never depends on PHP/MySQL being healthy to just prove the container is
alive).

---

## Phase D — MySQL service (steps 11-13)

### 11. MySQL task definition + service

Use [`ecs/mysql-task-definition.json`](../ecs/mysql-task-definition.json) in
this repo as the template — fill in your account ID, EFS filesystem ID, and
secret ARN.

```bash
aws ecs register-task-definition --cli-input-json file://ecs/mysql-task-definition.json

# Cloud Map namespace for service discovery (needed before Service Connect will work)
aws servicediscovery create-private-dns-namespace --name wordpress-staging --vpc $VPC_ID

aws ecs create-service \
  --cluster wordpress-cluster --service-name mysql-service \
  --task-definition mysql-task:1 --desired-count 1 --launch-type FARGATE --platform-version LATEST \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-<private-a>,subnet-<private-b>],securityGroups=[$MYSQL_SG],assignPublicIp=DISABLED}" \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "wordpress-staging",
    "services": [{"portName":"mysql","discoveryName":"mysql","clientAliases":[{"port":3306,"dnsName":"mysql"}]}]
  }'
```

**Watch out — a real, subtle CLI bug hit during this build:** running
`update-service` with both `--task-definition` and
`--service-connect-configuration` in the same call, *without* also passing
`--desired-count`, silently created a new deployment pinned at
**desired count 0** — it just sat there doing nothing, no error. Passing
`--desired-count` explicitly on every `update-service` call that also
changes the task definition or Service Connect config avoids this
completely. If a deployment's `runningCount` won't move off 0 and there's no
error in `events`, check whether its `desiredCount` is actually 0 first.

**Also watch out:** `aws ecs describe-services --query
'service.serviceConnectConfiguration'` looks like it should show your
Service Connect settings but doesn't — that field lives at
`service.deployments[].serviceConnectConfiguration`, one level down. Don't
conclude Service Connect failed just because the top-level field comes back
empty; check the deployment-level field before assuming it's broken.

**Why it matters:** this is the "MySQL as its own ECS service, not baked
into the WordPress task" design. WordPress reaches it as `mysql` — a stable
name that survives the MySQL task being replaced — instead of a hard-coded
IP that breaks the moment ECS reschedules the task onto a new ENI.

### 12. Initialize MySQL

Nothing to do here beyond what's already in the task definition — the
official `mysql:8.4` image initializes its own data directory from the
`MYSQL_DATABASE` / `MYSQL_USER` / `MYSQL_PASSWORD` / `MYSQL_ROOT_PASSWORD`
secrets on first boot. Watch the logs on first launch to confirm:

```bash
aws logs tail /aws/ecs/wordpress-staging --log-stream-name-prefix wordpress/mysql --follow
```

You want to see it get all the way to `ready for connections ... port: 3306`
— MySQL's entrypoint script actually starts a *temporary* internal server
first (port 0/socket-only) to apply the initial users, then restarts into
the real one on 3306. Seeing the first "ready for connections" isn't enough
on its own; wait for the second one with the real port number.

### 13. Initialize WordPress EFS

WordPress core files have to land on the EFS filesystem *somehow* before any
ECS task can serve them, and there's no ECS task yet at this point in the
build. Options:

- A throwaway Fargate task or EC2 instance with the EFS mounted, used once
  to copy files in, then torn down (or repurposed — this build reused an
  existing EC2 instance in the private subnet exactly for this, over SSM,
  no SSH involved).
- `aws efs` doesn't have a "copy files in" API — you need *something*
  with an NFS client attached to the filesystem.

```bash
# From a box with network access to the EFS mount target and nfs-common/amazon-efs-utils installed:
sudo mount -t nfs4 -o nfsvers=4.1 <fs-id>.efs.us-east-1.amazonaws.com:/ /mnt/wordpress-efs
curl -sSL -o /tmp/wp.tar.gz https://wordpress.org/latest.tar.gz
tar -xzf /tmp/wp.tar.gz -C /tmp
cp -r /tmp/wordpress/* /mnt/wordpress-efs/
chown -R 33:33 /mnt/wordpress-efs   # 33 = www-data, matches the PHP image's user
```

Then write a `wp-config.php` that reads DB credentials from environment
variables — **not** hardcoded values — because those variables are exactly
what the ECS task definition's `secrets` block injects at runtime:

```php
define( 'DB_NAME', getenv('DB_NAME') ?: 'wordpress' );
define( 'DB_USER', getenv('DB_USER') );
define( 'DB_PASSWORD', getenv('DB_PASSWORD') );
define( 'DB_HOST', getenv('DB_HOST') ?: 'mysql' );
```

**Why it matters:** get the file ownership wrong (e.g. leave it root-owned)
and PHP-FPM running as `www-data` won't be able to write `wp-content/uploads`
or manage its own cache — uploads will fail with a permissions error that has
nothing obviously to do with EFS.

---

## Phase E — WordPress images (steps 14-16)

### 14-15. Build the images

Use [`php/Dockerfile`](../php/Dockerfile) and
[`nginx/Dockerfile`](../nginx/Dockerfile) in this repo directly:

```bash
docker build -t wordpress-php:local ./php
docker build -t wordpress-nginx:local ./nginx
```

Test them locally before touching AWS at all — this repo's
[`docs/testing-results.md`](testing-results.md) shows exactly how (Nginx and
PHP-FPM sharing one Docker network namespace to mimic an ECS task's
`awsvpc` mode, MySQL on a separate container with a network alias standing
in for Service Connect). If it doesn't work locally with the exact same
images, it won't work in Fargate either — and it's much faster to debug on
your own machine.

### 16. Push images manually first

```bash
GIT_SHA=$(git rev-parse --short HEAD)
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com

docker tag wordpress-php:local <account>.dkr.ecr.us-east-1.amazonaws.com/wordpress-php:$GIT_SHA
docker push <account>.dkr.ecr.us-east-1.amazonaws.com/wordpress-php:$GIT_SHA
# same for nginx
```

**Watch out — the single most disruptive bug in this whole build:** if
you're building on Apple Silicon (or any ARM64 dev machine) and pushing
without `--platform`, you get an **arm64-only image manifest**. Fargate
defaults to x86_64 and will fail every task launch with:

```
CannotPullContainerError: ... image Manifest does not contain descriptor
matching platform 'linux/amd64'
```

Two fixes, pick one **and stay consistent about it everywhere** (task
definition *and* CI):
1. Build with `docker build --platform linux/amd64` to match Fargate's default, or
2. Set the task definition's `runtimePlatform` to
   `{"cpuArchitecture": "ARM64"}` and keep building for arm64 — this is what
   this build did, since Fargate ARM64 (Graviton) is also ~20% cheaper per
   vCPU/GB. If you go this route, your CI pipeline (GitHub Actions runners
   are amd64) needs to cross-build for arm64 explicitly — see
   [`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml)'s
   `docker/setup-qemu-action` + `docker/build-push-action` steps with
   `platforms: linux/arm64`. Forgetting this on the CI side reintroduces the
   exact same failure on the very next automated deploy.

Never tag or push `latest` as anything you actually deploy from — SHA tags
only, so every deployed image traces back to an exact commit and rollback
means "deploy the previous tag," not "hope you remember what changed."

---

## Phase F — WordPress service (steps 17-19)

### 17. WordPress task definition

Use [`ecs/wordpress-task-definition.json`](../ecs/wordpress-task-definition.json)
as the template. Key details worth noting in it:

- Two containers, one task: `nginx` (port 80) and `php` (port 9000) —
  they talk over `127.0.0.1:9000` because Fargate's `awsvpc` mode gives
  every container in a task the same network namespace.
- Both mount the same EFS volume at `/var/www/html`.
- `php`'s `DB_HOST` is the literal string `mysql` — resolved via Service
  Connect, never a hardcoded IP.
- `runtimePlatform` must match how you built your images (see step 16).

```bash
aws ecs register-task-definition --cli-input-json file://ecs/wordpress-task-definition.json
```

### 18. WordPress ECS service

```bash
aws ecs create-service \
  --cluster wordpress-cluster --service-name wordpress-service \
  --task-definition wordpress-task:1 --desired-count 1 --launch-type FARGATE --platform-version LATEST \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-<private-a>,subnet-<private-b>],securityGroups=[$ECS_SG],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=nginx,containerPort=80" \
  --service-connect-configuration '{"enabled": true, "namespace": "wordpress-staging"}' \
  --deployment-configuration 'minimumHealthyPercent=100,maximumPercent=200'
```

Note this service only needs Service Connect *enabled* (client mode) to
resolve `mysql` — it doesn't need its own `services` entry, since nothing
else needs to discover WordPress by a Service Connect name (the ALB reaches
it directly via the target group instead).

### 19. Verify the whole path

```bash
ALB_DNS=$(aws elbv2 describe-load-balancers --names wordpress-alb --query 'LoadBalancers[0].DNSName' --output text)
curl http://$ALB_DNS/health     # expect 200, "OK"
curl http://$ALB_DNS/           # expect the WordPress installer, or your installed site

aws elbv2 describe-target-health --target-group-arn $TG_ARN   # expect state: healthy
```

**Don't skip this manual verification to jump straight to CI/CD.** Everything
upstream of GitHub Actions in this design — networking, security groups,
IAM, service discovery, EFS — has to work independently of CI first.
Debugging a broken security group rule is much harder when it's tangled up
with also debugging a GitHub Actions workflow at the same time.

---

## Phase G — CI/CD (steps 20-24)

### 20. GitHub OIDC provider + role

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com

aws iam create-role --role-name GitHubActions-ECS-Deploy --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
      "StringLike": {"token.actions.githubusercontent.com:sub": "repo:<owner>/<repo>:environment:staging"}
    }
  }]
}'
```

Then a least-privilege policy: ECR push scoped to the two repos, ECS
register/describe/update scoped to the one service, `iam:PassRole` limited
to exactly the execution and task roles (never wider), CloudWatch Logs
scoped to the one log group. Never attach `AdministratorAccess` to make a
pipeline "just work."

**Why the `environment:staging` scoping matters:** without it, the trust
policy's `sub` condition would be `repo:<owner>/<repo>:ref:refs/heads/main`
or similar — meaning *any* workflow on that branch could assume the role.
Scoping to a GitHub Environment means only jobs that explicitly declare
`environment: staging` can ever get a token for it, and GitHub Environments
support their own protection rules (required reviewers, wait timers) on top.

### 21. GitHub environment + variables

```bash
gh api repos/<owner>/<repo>/environments/staging -X PUT
gh variable set AWS_REGION --env staging --body "us-east-1"
gh variable set ECS_CLUSTER --env staging --body "wordpress-cluster"
gh variable set ECS_SERVICE --env staging --body "wordpress-service"
gh variable set ECS_TASK_FAMILY --env staging --body "wordpress-task"
gh variable set ECR_NGINX_REPOSITORY --env staging --body "wordpress-nginx"
gh variable set ECR_PHP_REPOSITORY --env staging --body "wordpress-php"
gh secret set AWS_ROLE_ARN --env staging --body "arn:aws:iam::<account>:role/GitHubActions-ECS-Deploy"
```

**Watch out:** `ECS_TASK_FAMILY` is easy to skip if you're copying the
simplified example workflow from the original design doc — that example
downloads "the current task definition" using the *service* name, which
happens to work only by coincidence if your service and task family share a
name. They don't have to, and in this build they didn't
(`wordpress-service` running `wordpress-task`) — this exact mismatch was a
real bug caught by re-reading the workflow before its first run. Use a
dedicated variable for the task family, not the service name.

### 22-24. GitHub Actions workflow

Use [`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml)
directly. It does, in order: checkout, OIDC auth, ECR login, cross-build
both images for the platform your task definition expects, push with the
git SHA as the tag, download the current task definition by family name,
render the two new image URIs into it, register a new revision, update the
service, wait for stability, then hit `/health` through the ALB as a final
sanity check.

**Verify a run actually completes:**

```bash
gh run list -R <owner>/<repo> --limit 5
gh run view <run-id> -R <owner>/<repo> --log-failed   # if it failed
```

**If `Configure AWS credentials` fails with `Not authorized to perform
sts:AssumeRoleWithWebIdentity`** and the trust policy, OIDC provider, and
GitHub environment all check out correctly (they will, if you followed step
20 exactly): this can also happen on a **brand-new AWS account still in its
identity-verification window** — AWS gates some federated-identity
capabilities for up to ~2 days on new accounts, and this build hit exactly
that. Confirm by checking whether other "new account" features are also
gated (CloudShell showing "account verification is in progress" is the
tell). If so, there's nothing to fix in the config — just wait and retry
`gh workflow run deploy.yml -R <owner>/<repo>`.

**If `on: push` doesn't seem to trigger any run at all** on a repo you just
created, but `gh workflow run` (manual dispatch) works immediately: this
looks like a GitHub-side indexing delay on brand-new repositories rather
than a workflow configuration problem. Add a `workflow_dispatch: {}` trigger
so you can test the pipeline itself without waiting on push-event
propagation, and don't assume your YAML is broken just because `on: push`
stayed quiet for the first commit or two.

---

## Phase H — Observability (steps 25-26)

### 25. CloudWatch logs

```bash
aws logs create-log-group --log-group-name /aws/ecs/wordpress-staging
aws logs put-retention-policy --log-group-name /aws/ecs/wordpress-staging --retention-in-days 14
```

Point every container's `logConfiguration` at this one group with a
distinct `awslogs-stream-prefix` (`wordpress/nginx`, `wordpress/php`,
`wordpress/mysql`) rather than one log group per task family — makes
`aws logs tail --log-stream-name-prefix wordpress/nginx --follow` work
cleanly across every task that's ever run, current or past.

**Why it matters:** set a retention period. The default is "never expire,"
which is a slow, silent cost leak on a long-running app.

### 26. Health checks

Three layers, each catching a different failure mode:

- **Nginx's own `/health` endpoint** (`nginx/default.conf`) — returns a bare
  `200 OK` with no PHP/MySQL dependency, so it only ever fails if Nginx
  itself is actually down.
- **ECS container health check** (in the task definition) — runs
  `curl -f http://localhost/health` from inside the container; this is what
  lets ECS itself notice and replace an unhealthy task before the ALB does.
- **ALB target group health check** — same path, checked externally; this
  is what actually controls whether the ALB routes real traffic to a task.

**Why three:** a container can be "running" from ECS's perspective while
Nginx has crashed inside it — the container-level health check catches
that. A task can be healthy but still not receiving traffic if the ALB
hasn't finished registering it yet — the target-group health check is what
gates real traffic. Skip the container-level check and ECS will happily
leave a broken task running until something else notices.

---

## Phase I — Production hardening (steps 27-30)

These are staging-to-production gaps, not staging bugs. Full detail in
[`production-gap-analysis.md`](production-gap-analysis.md); summary here:

### 27. Autoscaling

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs --resource-id service/wordpress-cluster/wordpress-service \
  --scalable-dimension ecs:service:DesiredCount --min-capacity 1 --max-capacity 3

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs --resource-id service/wordpress-cluster/wordpress-service \
  --scalable-dimension ecs:service:DesiredCount --policy-name wordpress-cpu-target-tracking \
  --policy-type TargetTrackingScaling --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"},
    "ScaleOutCooldown": 60, "ScaleInCooldown": 120
  }'
```

Kept minimum capacity at 1 in this build rather than the "2 for production"
suggestion — that decision trades some resilience for not permanently
doubling Fargate cost on a credit-limited staging account. Raise it to 2 (and
spread across the 2 AZs) once real traffic or real uptime requirements
justify the cost.

### 28. HTTPS/ACM

Needs a real domain — ACM cannot issue or validate a certificate against a
raw `*.elb.amazonaws.com` address. Once you have one:

```bash
aws acm request-certificate --domain-name staging.example.com --validation-method DNS
# add the returned DNS validation record, wait for ISSUED, then:
aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTPS --port 443 \
  --certificates CertificateArn=<cert-arn> --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### 29. WAF

```bash
aws wafv2 create-web-acl --name wordpress-waf --scope REGIONAL --default-action Allow={} \
  --rules '[{"Name":"AWS-AWSManagedRulesWordPressRuleSet","Priority":0,"OverrideAction":{"None":{}},"Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesWordPressRuleSet"}},"VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"wordpress-waf"}}]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=wordpress-waf
aws wafv2 associate-web-acl --web-acl-arn <arn> --resource-arn $ALB_ARN
```

Has an ongoing cost (~$5-10/month base plus per-request charges) — worth it
once there's real traffic to protect; skip it on a bare staging environment
with no public audience yet.

### 30. Everything else

Aurora/RDS migration, Route 53, Multi-AZ minimum-2 tasks, automated backups,
monitoring/alarms, VPC endpoints, container hardening (`readonlyRootFilesystem`,
dropped capabilities) — each with its own section and reasoning in
[`production-gap-analysis.md`](production-gap-analysis.md). None of these
block a staging environment from working; all of them matter before calling
it production.
