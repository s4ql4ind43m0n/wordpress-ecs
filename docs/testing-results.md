# Testing Results

Each test records what was checked, how, and the actual result observed during
this build. Local tests were run before anything was deployed to AWS, per the
"prove it manually before automating it" approach in the source design doc.

## Local container tests (pre-AWS)

Environment: Docker Desktop on macOS. `wordpress-nginx:local` and
`wordpress-php:local` built from this repo's Dockerfiles (unmodified from
what ships to ECR). Nginx and PHP-FPM were run with `--network container:`
so they share one network namespace, the same way two containers in one ECS
task share `localhost` — the closest local approximation of Fargate's
`awsvpc` networking. MySQL ran as a separate container on a user-defined
bridge network with a `mysql` network alias, standing in for ECS Service
Connect's stable service name.

| # | Test | Method | Result |
|---|------|--------|--------|
| 1 | PHP-FPM image builds | `docker build ./php` | **PASS** — builds clean, no errors |
| 2 | Nginx image builds | `docker build ./nginx` | **PASS** — builds clean, no errors |
| 3 | Nginx `/health` | `curl http://localhost:8080/health` | **PASS** — `HTTP 200`, body `OK` |
| 4 | Nginx -> PHP-FPM (`127.0.0.1:9000`) | Request `/` (PHP-rendered) | **PASS** — PHP executed, WordPress responded (not a static/502 page) |
| 5 | PHP -> MySQL connectivity | Same request; initially failed | **FAIL then PASS** — see note below |
| 6 | WordPress installation | POST to `wp-admin/install.php?step=2` (title, admin user, password, email) | **PASS** — `Success!`, homepage then served `<title>WordPress ECS Local Test</title>` |
| 7 | MySQL persistence after container replacement | `docker rm -f` the MySQL container, recreated with the same named volume (`wp-test-mysql-data:/var/lib/mysql`), reloaded homepage | **PASS** — site still showed the installed title; no re-install prompt |
| 8 | WordPress file persistence after "task" replacement | Wrote a marker file into `wp-content/uploads/`, `docker rm -f` on both Nginx and PHP containers, recreated them against the same shared volume | **PASS** — marker file still served at the same URL; homepage still loaded |

**Note on test 5:** the first attempt returned `HTTP 500` / "Database Error".
Root cause: `wp-config.php` used `DB_HOST=mysql` (the intended stable service
name, matching the "don't hard-code the MySQL IP" guidance in the design doc),
but the MySQL container was only reachable under its actual container name.
Fixed by giving the MySQL container a Docker network alias (`--network-alias
mysql`) — the direct local equivalent of what ECS Service Connect provides in
AWS. This is exactly the class of bug the doc's service-discovery
recommendation exists to avoid; it's now verified working, and the same
`DB_HOST=mysql` value carries over unchanged to the ECS task definition.

## AWS-environment tests

Run against the real `wordpress-staging-vpc` / `wordpress-cluster` on
2026-08-26. See [implementation-report.md](implementation-report.md) for the
full list of what already existed vs. what was fixed or created this session.

| # | Test | Method | Result |
|---|------|--------|--------|
| 1 | ALB `/health` reachable from the internet | `curl http://<alb-dns>/health` | **PASS** — `HTTP 200`, body `OK` |
| 2 | ALB target group shows targets `HEALTHY` | `aws elbv2 describe-target-health` | **PASS** |
| 3 | Full path: Internet -> ALB -> ECS Nginx -> PHP-FPM -> MySQL ECS service | `curl http://<alb-dns>/` | **PASS** — reached the WordPress installer, then a real installed site, entirely through the ALB |
| 4 | WordPress installation via the live ALB endpoint | POST to `wp-admin/install.php?step=2` | **PASS** — `Success!`, homepage served `<title>WordPress ECS Staging</title>` |
| 5 | MySQL persistence after a real ECS task replacement | `aws ecs stop-task` on the running mysql task; service replaced it | **PASS** — site still showed the installed title after the new task came up, on the same EFS-backed volume |
| 6 | WordPress file persistence after a real ECS task replacement | `aws ecs stop-task` on the running wordpress task | **PASS** — new task registered healthy in the target group; homepage and `/health` both fine |
| 7 | ECS Service Connect (`DB_HOST=mysql`) | Live task env resolves `mysql` via the `wordpress-staging` Cloud Map namespace | **PASS** — no hardcoded IP anywhere; confirmed by test 3/4 succeeding |
| 8 | CloudWatch log groups receiving nginx/php/mysql logs | `aws logs tail /aws/ecs/wordpress-staging --log-stream-name-prefix ...` | **PASS** — real request/response lines visible for all three containers |
| 9 | Rollback to a previous task definition revision | Deployed `wordpress-task:3`, confirmed serving, rolled back to `wordpress-task:2` | **PASS** — target went healthy again on the older revision, site unaffected |
| 10 | GitHub Actions: OIDC auth, build, push, new task definition revision, deploy | Manually dispatched `deploy.yml` (the `on: push` trigger itself did not fire — see note) | **BLOCKED** — see note below |

**Note on test 10:** every AWS-side piece for CI/CD is provisioned and correctly
configured (OIDC provider, `GitHubActions-ECS-Deploy` role with a trust policy
scoped to `repo:s4ql4ind43m0n/wordpress-ecs:environment:staging`, least-privilege
permissions, GitHub `staging` environment with `AWS_ROLE_ARN` secret and all
required variables). The `Configure AWS credentials` step still fails with
`Not authorized to perform sts:AssumeRoleWithWebIdentity` even after the trust
policy was temporarily widened to the whole repository (ruling out a `sub`-claim
mismatch) and after waiting for OIDC provider propagation. This account showed
"account verification is in progress... may take up to two days" when CloudShell
was attempted earlier the same session, and federated `AssumeRoleWithWebIdentity`
calls from a brand-new OIDC provider are a commonly-gated capability during that
same hold — the two failures line up. No code or policy change is expected to
fix this; re-run the workflow (`gh workflow run deploy.yml`) after the account
finishes verifying. Also worth noting separately: the very first two pushes to
`main` did not register a workflow run at all via the `on: push` trigger (`gh
workflow run` / `workflow_dispatch` worked immediately) — this looks like a
GitHub-side indexing quirk on a just-created repository rather than anything
in this repo's configuration; worth a plain push once account verification
clears, just to confirm `on: push` is firing normally by then too.

**Retried 2026-08-26, ~1 hour later:** same result, same error
(`Not authorized to perform sts:AssumeRoleWithWebIdentity`), confirming this
is a time-based account hold rather than a transient blip. No further retries
attempted this session — re-run `gh workflow run deploy.yml -R
s4ql4ind43m0n/wordpress-ecs` after the account finishes verifying (AWS's own
message said up to 2 days).

## Additional changes verified this session (redo pass)

| # | Test | Method | Result |
|---|------|--------|--------|
| 11 | Autoscaling target and policy registered | `aws application-autoscaling describe-scalable-targets` / `describe-scaling-policies` | **PASS** — min 1 / max 3, target-tracking on 70% CPU utilization, confirmed in AWS's response |
| 12 | HTTPS/ACM | N/A | **DEFERRED BY DECISION** — no domain available; ACM cannot issue/validate a cert against a raw ALB DNS name, so this is a hard blocker, not an oversight |
| 13 | WAF | N/A | **DEFERRED BY DECISION** — skipped to avoid adding ~$5-10/month to an account with $27.48 remaining credit |
