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

## AWS-environment tests (pending)

These require the live AWS account and are not yet run — see
[implementation-report.md](implementation-report.md) for current provisioning
status:

- [ ] ALB `/health` reachable from the internet
- [ ] ALB target group shows targets `HEALTHY`
- [ ] Full path: Internet -> ALB -> ECS Nginx -> PHP-FPM -> MySQL ECS service
- [ ] WordPress file persistence after a real ECS task replacement
- [ ] MySQL persistence after a real ECS task replacement
- [ ] GitHub Actions: OIDC auth, build, push, new task definition revision, deploy
- [ ] CloudWatch log groups receiving nginx/php/mysql logs
- [ ] Rollback to a previous task definition revision
