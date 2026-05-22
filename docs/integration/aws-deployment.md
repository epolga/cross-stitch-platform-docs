# Contract: AWS Deployment

## 1. Contract Name

**AWS deployment — Elastic Beanstalk, EC2, S3, CloudFront, region** — the operational contract that pins the cross-stitch web app to its production Elastic Beanstalk environment (`cross-stitch-com-env-clone` in `us-east-1`), the EC2 instances behind it, the `cross-stitch-designs` S3 bucket fronted by the CloudFront distribution `d2o1uvvg91z7o4.cloudfront.net`, and the operator workflow that deploys, restarts, and reboots them. There is no CI; deployments are driven from the operator workstation via `eb deploy`, and post-upload restarts are driven from the Uploader's "Restart Elastic Beanstalk" button (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:366-394`, also invoked as step 5 of `RunFullUploadFlowAsync` at `MainWindow.xaml.cs:664-675`).

## 2. Purpose

The cross-stitch platform runs entirely on AWS infrastructure that is configured by hand in a single operator's account. The Uploader (a Windows WPF tool) and the cross-stitch Next.js web app share the same region, the same DynamoDB tables, the same S3 design bucket, and the same Elastic Beanstalk environment name — but **none of those names lived in documentation** until now (per `d:/ann/Git/cross-stitch-platform-docs/CLAUDE.md:50` "AWS deployment assumptions" is on the `do-not-invent` list). This contract codifies the names, the operator commands, and the failure modes so a new operator (or a future Olga) can verify the live environment against what the two code repos assume.

This document deliberately does **not** repeat the schemas already captured in the sibling contracts; cross-references go to:
- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/dynamodb-schema.md` for tables `CrossStitchItems`, `CrossStitchUsers`, `PasswordResetTokens`, `SubscriptionEvents`.
- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/s3-paths.md` for S3 key shapes and CloudFront URL templates.
- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/pinterest-metadata.md` for the unrelated Pinterest token / OAuth surface that happens to share the same AWS account.

## 3. Scope

In scope:
- AWS account, region, and IAM-role identity used by both the web app at runtime and the Uploader at write time.
- Elastic Beanstalk application `cross-stitch-com`, environment `cross-stitch-com-env-clone`, and the `.ebextensions/*.config` files that configure it (`d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:1-10`, `d:/ann/Git/cross-stitch/.ebextensions/04_options.config:1-4`).
- The saved configuration snapshot `eb-configuration-2025-12-12.cfg.yml` and its drift-from-live risk (`d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:1-89`).
- EC2 instances launched by EB, tagged `Name = cross-stitch-com-env-clone`, and the Uploader-side helper that reboots them (`d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs:25-43`).
- CloudFront distribution `d2o1uvvg91z7o4.cloudfront.net`, in its role as the **deployment-time public face** of the design-asset bucket (the URL-shape contract itself is owned by `s3-paths.md`).
- Operator restart procedures: `eb deploy`, Uploader "Restart Elastic Beanstalk" button (`MainWindow.xaml.cs:366-394`), automatic post-upload restart (`MainWindow.xaml.cs:664-675`), and per-instance EC2 reboot (`EC2Helper.cs:74-115`).

Out of scope:
- DynamoDB attribute-level schema (see `dynamodb-schema.md`).
- S3 key shapes, photo/PDF/chart templates, CloudFront URL templates (see `s3-paths.md`).
- Pinterest OAuth / token storage (see `pinterest-metadata.md`).
- The Uploader's own deployment — it has none; it is a local WPF tool that ships as a developer-built `bin/Release` folder (`d:/ann/Git/cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:226-227`).
- The cross-stitch CI pipeline — it has none; tests/builds are run locally and the Jenkinsfile was removed (`platform-architecture-summary.md:58, 220`).

## 4. Data Formats

### 4.1 AWS account and region

| Field | Value | Source |
|---|---|---|
| AWS account ID | `358174257684` | `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md:7`; embedded in `ServiceRole: arn:aws:iam::358174257684:role/aws-elasticbeanstalk-service-role` (`d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:60`); embedded in `SSLCertificateArns: arn:aws:acm:us-east-1:358174257684:...` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:35`) |
| Region | `us-east-1` | `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:7` (`default_region: us-east-1`); `d:/ann/Git/cross-stitch/.ebextensions/04_options.config:3` (`AWS_REGION: us-east-1`); `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:15` |
| Uploader runtime region | `RegionEndpoint.USEast1` | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:56` (EB helper); `MainWindow.xaml.cs:60` (S3 helper); region literal is **hardcoded in C#**, not read from `App.config` |

### 4.2 Elastic Beanstalk application + environment

| Field | Value | Source |
|---|---|---|
| EB application | `cross-stitch-com` | `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:6` (`application_name: cross-stitch-com`) |
| EB environment (live) | `cross-stitch-com-env-clone` | `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:3` (`main:` branch default); `d:/ann/Git/Uploader/Uploader/App.config:17` (`ElasticBeanstalkEnvironmentName`); `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:57` (C# fallback literal); `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md:12` |
| EB platform | `64bit Amazon Linux 2023 / Node.js 20 / 6.7.0` | `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:5-7` (`PlatformArn: arn:aws:elasticbeanstalk:us-east-1::platform/Node.js 20 running on 64bit Amazon Linux 2023/6.7.0`); `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md:13` |
| Listening port | `3000` | `d:/ann/Git/cross-stitch/.ebextensions/01_node.config:3` (`Port: 3000`); `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:17` (`PORT: '3000'`) |
| Environment type | Load-balanced (ALB) | `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:60-62` (`LoadBalancerType: application`); `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md:14` |
| Instance type | `t2.small` | `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:48-49` |
| EC2 key pair (saved) | `aws-eb2` | `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:82`. Note: `.elasticbeanstalk/config.yml:8` declares `default_ec2_keyname: aws-eb` — the saved snapshot and the live EB CLI default **disagree** (probable harmless drift; only matters if someone runs `eb create` from the CLI without `--cfg`). |
| Deployment policy | `AllAtOnce` | `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:9-11`. No rolling deploy — every push briefly takes the single instance offline. |
| Managed platform updates | Disabled | `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:44-47` (`ManagedActionsEnabled: false`, `InstanceRefreshEnabled: false`). |

### 4.3 EB application-environment variables (from `.ebextensions/04_options.config` + saved snapshot)

`.ebextensions/04_options.config` is the only `.ebextensions` file that mutates app config; it is checked into git and applied on every `eb deploy`:

```
option_settings:
  aws:elasticbeanstalk:application:environment:
    AWS_REGION: us-east-1
    DYNAMODB_TABLE_NAME: CrossStitchItems
```
(`d:/ann/Git/cross-stitch/.ebextensions/04_options.config:1-4`)

The remaining environment variables — including secrets — are **not in git** and live only on the live environment (visible via `eb printenv` or in the saved snapshot, which is itself a frozen copy):

| Variable | Value (snapshot) | Source line |
|---|---|---|
| `AWS_REGION` | `us-east-1` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:15` |
| `DYNAMODB_TABLE_NAME` | `CrossStitchItems` | `:23` |
| `DDB_USERS_TABLE` | `CrossStitchUsers` | `:19` |
| `DDB_RESET_TOKENS_TABLE` | `PasswordResetTokens` | `:22` |
| `S3_BUCKET_NAME` | `cross-stitch-sitemap-cache` | `:18` (note: this is the **sitemap-cache bucket**, *not* the design bucket — design assets are read from CloudFront, not S3 directly; cross-ref `s3-paths.md` §4.1) |
| `PORT` | `'3000'` | `:17` |
| `NODE_ENV` | `production` | `:26` |
| `NEXT_PUBLIC_BASE_URL` | `https://www.cross-stitch-pattern.net` | `:14` (note: differs from cross-stitch.com; the EB env serves both apex domains per `ENVIRONMENT-SETUP.md:48-57`) |
| `AWS_SES_FROM_EMAIL` | `ann@cross-stitch-pattern.net` | `:21` |
| `ADMIN_EMAIL` | `olga.epstein@gmail.com` | `:27` |
| `PAYPAL_API_URL` | `https://api-m.paypal.com` | `:13` |
| `PAYPAL_CLIENT_ID` / `NEXT_PUBLIC_PAYPAL_CLIENT_ID` | `AQin-IMhxWJE8DZwOQRz7w2rVPltgJ6RJ7Z2gs8vaGbSzccSfKwK8swYNGYUzZjiyTc7Bf3YGWBcqdrj` | `:16, :20` |
| `PAYPAL_CLIENT_SECRET` | `secret` (literal string in snapshot — placeholder; live value is set out-of-band) | `:25` |
| `PAYPAL_WEBHOOK_ID` | `03297922HC292773V` | `:24` |

The other top-level `.ebextensions/*.config` files configure the host rather than the app:

| File | Purpose |
|---|---|
| `01_build.config` | Runs `npm install` as a container_command; sets `NODE_ENV: production` (`d:/ann/Git/cross-stitch/.ebextensions/01_build.config:1-6`) |
| `01_node.config` | Pins port `3000`, sets default ALB health-check path `/` (`d:/ann/Git/cross-stitch/.ebextensions/01_node.config:1-8`) |
| `02_swap.config` | Allocates a 1 GiB swap file (because `t2.small` has only 2 GiB RAM) (`d:/ann/Git/cross-stitch/.ebextensions/02_swap.config:1-5`) |
| `05_health_check.config` | Overrides health-check path to `/api/health` (`d:/ann/Git/cross-stitch/.ebextensions/05_health_check.config:1-3`) — note `01_node.config:5` also sets `HealthCheckPath: /`; load order of `.ebextensions` files is alphabetical, so `05_*` wins. |
| `06_cloudwatch_logs.config` | Installs the CloudWatch agent and streams `/var/log/myapp/webhook.log` to log group `/aws/elasticbeanstalk/cross-stitch-env/var/log/myapp/webhook.log` (`d:/ann/Git/cross-stitch/.ebextensions/06_cloudwatch_logs.config:1-40`). Note: the log-group name hardcodes `cross-stitch-env`, not the live env name `cross-stitch-com-env-clone` — historical name drift, low impact. |
| `07_ntp.config` | Enables `chronyd` (`d:/ann/Git/cross-stitch/.ebextensions/07_ntp.config:1-4`) |

### 4.4 Networking, TLS, and IAM (from saved snapshot)

| Field | Value | Source |
|---|---|---|
| VPC | `vpc-0ce5d2e526673f056` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:79` |
| Instance subnets | `subnet-0c10a08cf49f2c0ba, subnet-03286dbc21b9904be, subnet-082aefa478054687c, subnet-0f98bf5d93b8e84b2, subnet-061daa49ea56c0228, subnet-06944f2885281f1c0` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:29` |
| ELB subnets | same set | `saved_configs/eb-configuration-2025-12-12.cfg.yml:39` |
| Public IPs | `AssociatePublicIpAddress: true` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:30-31` |
| HTTPS listener | `443`, `Protocol: HTTPS`, security policy `ELBSecurityPolicy-2016-08` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:32-37`. Note: `ENVIRONMENT-SETUP.md:69-70` documents `ELBSecurityPolicy-TLS13-1-2-2021-06` — **the saved snapshot and the production doc disagree**; either the snapshot is stale or the documentation aspirational. |
| ACM certificate | `arn:aws:acm:us-east-1:358174257684:certificate/8c0f050c-8a28-442f-83ed-e496eda2f04f` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:35`. SANs (per `ENVIRONMENT-SETUP.md:74-78`): `cross-stitch.com`, `www.cross-stitch.com`, `cross-stitch-pattern.net`, `www.cross-stitch-pattern.net`. |
| EB service role | `arn:aws:iam::358174257684:role/aws-elasticbeanstalk-service-role` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:60` |
| EC2 instance profile | `aws-elasticbeanstalk-ec2-role` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:81`. The actual policy attachments on this role are **not in git**; runtime AWS SDK calls from the Next.js app (DynamoDB, SES, S3 reads) depend on whatever inline policies this role carries — see §10. |
| Health reporting | `enhanced` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:77` |
| EB logs to CloudWatch (StreamLogs) | `false` | `saved_configs/eb-configuration-2025-12-12.cfg.yml:58` — only the webhook-log custom stream from `06_cloudwatch_logs.config` is shipped. |

### 4.5 CloudFront

- Distribution domain: `d2o1uvvg91z7o4.cloudfront.net` (referenced from `d:/ann/Git/cross-stitch/next.config.js:12`).
- Origin: `cross-stitch-designs` S3 bucket (inferred; the bucket is named at `d:/ann/Git/Uploader/Uploader/App.config:4` and `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:60`).
- Whitelisted path in Next.js `images.remotePatterns`: **only** `/photos/**` (`d:/ann/Git/cross-stitch/next.config.js:9-15`).
- PDFs at `/pdfs/**` are served by the same distribution but are loaded as direct browser GETs, not via `next/image`. CloudFront is not signed; objects are public-read.
- Distribution config, OAI/OAC, behaviors, TTLs, and invalidation policy are **not in git** in either repo.

### 4.6 EC2 instances behind EB

EB tags each instance with `Name = <environment-name>`. The Uploader uses this tag to enumerate the fleet:

- `d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs:30` — `await GetInstanceIdsByTagAsync(_ec2Client, "Name", _environmentName);`
- `d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs:53-69` — filters by `tag:Name = cross-stitch-com-env-clone`, excludes Terminated and Stopped.

Health-check expectation (per `EC2Helper.IsInstanceRestartedAndHealthyAsync`, `EC2Helper.cs:156-220`): instance is `Running`, AWS `SystemStatus` is `Ok`, and AWS `InstanceStatus` is `Ok`. Polling is fixed at 5 s intervals × 60 attempts = 5-minute maximum wait (`EC2Helper.cs:132-145`).

## 5. API Endpoints / Interfaces

There is no HTTP API surface in this contract; the interfaces are operator-driven AWS commands and Uploader-driven AWS SDK calls.

### 5.1 Operator interface: `eb deploy` from the developer machine

- The cross-stitch repo is wired to the EB CLI via `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml`. Branch default: `main → cross-stitch-com-env-clone`.
- Documented procedure: `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md:30-42` (PowerShell `eb init`, `eb create <env> --cfg eb-configuration-2025-12-12`, `eb status`, `eb health`).
- **There is no CI.** No GitHub Actions, no Jenkinsfile (`d:/ann/Git/cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:58, 220`). Every production push happens by running `eb deploy` from Olga's workstation against the local `main` branch.

### 5.2 Uploader interface: `RestartAppServer` via AWS SDK

The Uploader includes a "Restart Elastic Beanstalk" button. Two call paths land in the same helper:

| Trigger | Call site | Notes |
|---|---|---|
| Manual UI button | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml:119` (`Click="BtnRestartEb_Click"`) → `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:366-394` | Disables the button while in flight; logs progress and result to `txtStatus`. |
| Automatic post-upload | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:664-675` (step 5 of `RunFullUploadFlowAsync`) | Runs after every successful chart/PDF/photo upload and DynamoDB insert; before the EB restart finishes, the operator's UI is already returning. |

Both paths construct `ElasticBeanstalkHelper` once at startup:
```
private readonly ElasticBeanstalkHelper _elasticBeanstalkHelper =
    new ElasticBeanstalkHelper(
        RegionEndpoint.USEast1,
        ConfigurationManager.AppSettings["ElasticBeanstalkEnvironmentName"] ?? "cross-stitch-com-env-clone");
```
(`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:54-57`)

Inside the helper, the SDK call is `RestartAppServerAsync`:
```
await _elasticBeanstalkClient.RestartAppServerAsync(
    new RestartAppServerRequest { EnvironmentName = environment.EnvironmentName }).ConfigureAwait(false);
```
(`d:/ann/Git/Uploader/Uploader/Helpers/ElasticBeanstalkHelper.cs:54-58`)

A pre-flight `DescribeEnvironmentsAsync` is issued first (`ElasticBeanstalkHelper.cs:34-48`); if the named env is not found, the helper returns `false` and logs `Elastic Beanstalk environment '<name>' was not found.` to the status callback. This is the *only* defense against env-name drift between `App.config:17` and the live AWS console.

### 5.3 Uploader interface: per-instance EC2 reboot by Name tag

`EC2Helper.RebootInstancesRequest` (`d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs:25-43`) is the lower-level escape hatch: it lists EC2 instances tagged `Name = cross-stitch-com-env-clone`, calls `RebootInstancesAsync` (`EC2Helper.cs:74-115`), waits for `Running` (`EC2Helper.cs:120-151`), then checks `SystemStatus == Ok && InstanceStatus == Ok` (`EC2Helper.cs:156-220`). This helper exists in code but **is not wired to any button in `MainWindow.xaml`** (greppable: no `EC2Helper` consumer in `MainWindow.xaml.cs`). It is dead-code-with-intent: callable from a future button, currently unreachable from the running app.

### 5.4 Runtime AWS SDK interface (via EC2 instance role)

The Next.js app running on EB makes outbound AWS calls using the instance role `aws-elasticbeanstalk-ec2-role`. The call surfaces are:
- DynamoDB read/write — table names are passed in via the `04_options.config` and snapshot env vars (see §4.3); cross-ref `dynamodb-schema.md`.
- S3 read for the sitemap cache (`S3_BUCKET_NAME: cross-stitch-sitemap-cache`); design assets are **not** accessed by the web app directly — they go through CloudFront unauthenticated.
- SES (`AWS_SES_FROM_EMAIL: ann@cross-stitch-pattern.net`) — transactional and password-reset email.
- The role-policy ARNs themselves are not in git.

### 5.5 Uploader credential interface

The Uploader instantiates `AmazonElasticBeanstalkClient`, `AmazonEC2Client`, `AmazonS3Client`, and `AmazonSimpleEmailServiceClient` with **no explicit credentials** (`d:/ann/Git/Uploader/Uploader/Helpers/ElasticBeanstalkHelper.cs:20`, `d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs:21`, `d:/ann/Git/Uploader/Uploader/Helpers/S3Helper.cs:20`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:50-51`). The AWS SDK default credential chain is used; on a developer machine that resolves to `%USERPROFILE%\.aws\credentials` or environment variables — **out of git, out of `App.config`**.

## 6. Versioning

**Unversioned (implicit).**

- The EB application has no concept of "v1"; deployments are sequential application versions identified by EB's auto-generated label (the `app-YYYYMMDD-HHMMSS` form). Rollback is not coded anywhere — see §9.
- The saved configuration `eb-configuration-2025-12-12.cfg.yml` is a **frozen snapshot from 2025-12-12** (file name + `DateCreated: '1765549610000'` ≈ 2025-12-12T13:46Z, `saved_configs/eb-configuration-2025-12-12.cfg.yml:3`). The live environment may have drifted since: any change made through the AWS console or `eb config` without a subsequent `eb config save` is lost from git.
- Recommended cadence: after any console-side EB change, re-run `eb config save eb-configuration-<YYYY-MM-DD>` and commit; or periodically diff via `aws elasticbeanstalk describe-configuration-settings --application-name cross-stitch-com --environment-name cross-stitch-com-env-clone` (see §11) and re-snapshot if material differences appear.
- The CloudFront distribution domain `d2o1uvvg91z7o4.cloudfront.net` is treated as **permanent**: it is hardcoded in `next.config.js:12`, `data-access.ts`, `DownloadPdfLink.tsx`, `RegisterForm.tsx`, and `PatternLinkHelper.cs` (see `s3-paths.md`). Changing it requires a coordinated multi-file release in two repos.
- The EB environment name `cross-stitch-com-env-clone` carries the `-clone` suffix from a prior environment swap; the name is now sticky and any rename requires updating at least `App.config:17`, `MainWindow.xaml.cs:57`, `ENVIRONMENT-SETUP.md:12`, and `.elasticbeanstalk/config.yml:3` together.

## 7. Ownership & Contacts

- **Maintainer:** Olga (`olga.epstein@gmail.com`, GitHub `epolga`). Same person owns AWS, both repos, and the operator workstation that runs `eb deploy`.
- **Code owner:** the `cross-stitch` repo holds the deployment manifest (`.elasticbeanstalk/`, `.ebextensions/`, `saved_configs/`, `ENVIRONMENT-SETUP.md`). The Uploader repo holds the consumer of the deployed env name (`App.config:17`, `MainWindow.xaml.cs:54-60`). When the env name changes, the change originates in `cross-stitch/.elasticbeanstalk/config.yml` and must be mirrored in `Uploader/App.config`.
- **AWS account owner:** `358174257684` is single-tenant for this project (`ENVIRONMENT-SETUP.md:7`).
- **`ADMIN_EMAIL` for runtime notifications:** `olga.epstein@gmail.com` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:27`).
- **`AWS_SES_FROM_EMAIL`:** `ann@cross-stitch-pattern.net` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:21`).

## 8. Dependencies

- **DynamoDB tables.** `CrossStitchItems`, `CrossStitchUsers`, `PasswordResetTokens` (`04_options.config:4` and `saved_configs/eb-configuration-2025-12-12.cfg.yml:19, 22-23`). Schema details in `d:/ann/Git/cross-stitch-platform-docs/docs/integration/dynamodb-schema.md`.
- **S3 buckets.** `cross-stitch-designs` (design assets, Uploader-writes / CloudFront-reads; named at `d:/ann/Git/Uploader/Uploader/App.config:4`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:60`) and `cross-stitch-sitemap-cache` (web-app artifact bucket; named at `saved_configs/eb-configuration-2025-12-12.cfg.yml:18`). Key shapes in `d:/ann/Git/cross-stitch-platform-docs/docs/integration/s3-paths.md`.
- **EB-logs bucket.** `elasticbeanstalk-us-east-1-358174257684` (per `ENVIRONMENT-SETUP.md:142-143`) — auto-created by EB; not referenced from code.
- **CloudFront distribution.** `d2o1uvvg91z7o4.cloudfront.net` (`next.config.js:12`). Origin is `cross-stitch-designs`; ownership entirely in the AWS console.
- **AWS SES.** `ann@cross-stitch-pattern.net` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:21`); used both by the web app at runtime (`d:/ann/Git/cross-stitch/src/lib/email-service.ts`, `src/lib/password-reset-email.ts` per `platform-architecture-summary.md:44, 203`) and by the Uploader for newsletter blasts (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:51`).
- **Route 53 hosted zones.** `cross-stitch-pattern.net` and `cross-stitch.com`, with ALIAS/CNAME records pointing at the ALB created by the EB env (`ENVIRONMENT-SETUP.md:46-58`). Not in git.
- **AWS IAM.** Service role `aws-elasticbeanstalk-service-role` and instance role `aws-elasticbeanstalk-ec2-role` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:60, 81`). Policy contents not in git.
- **AWS ACM certificate.** `arn:aws:acm:us-east-1:358174257684:certificate/8c0f050c-8a28-442f-83ed-e496eda2f04f` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:35`).
- **Operator tooling.** EB CLI (`eb` v3.x), AWS CLI (`aws`), and PowerShell on the operator workstation; documented in `ENVIRONMENT-SETUP.md:25-29`.
- **Default credential chain.** AWS SDK clients in the Uploader resolve credentials from `%USERPROFILE%\.aws\credentials` or env vars (see §5.5); the credential file itself is **not** referenced from any code in either repo.

## 9. Error Handling

Observed in code, not theoretical:

- **No rollback procedure.** A failed `eb deploy` can leave the env in a mixed state (e.g. dependencies installed but the new app version not yet healthy). `ENVIRONMENT-SETUP.md` does not document a `eb deploy --version <prior>` rollback path, and `DeploymentPolicy: AllAtOnce` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:11`) means every deploy briefly takes the single `t2.small` instance offline.
- **Uploader EB-restart errors are swallowed into the UI log.** `ElasticBeanstalkHelper.RestartEnvironmentAsync` catches `AmazonElasticBeanstalkException` (`Helpers/ElasticBeanstalkHelper.cs:63-66`) and generic `Exception` (`Helpers/ElasticBeanstalkHelper.cs:68-71`), writes `Elastic Beanstalk error: <msg>` to the callback, and returns `false`. The caller (`MainWindow.xaml.cs:374-385`) appends a one-line `Elastic Beanstalk restart failed.` to `txtStatus`. The upload pipeline as a whole still considers itself successful — the EB restart is fire-and-log, not a failure stop.
- **Auto-restart fires before instance is back.** `RestartAppServerAsync` is asynchronous in EB itself: the SDK call returns "submitted" while the actual restart takes ~30-60s. The Uploader code awaits only the SDK submission (`ElasticBeanstalkHelper.cs:54-60`), not the resulting `Updating → Ready` transition. If a second upload runs immediately, it lands during the restart window.
- **EC2 reboot helper has hard-coded poll budget.** `WaitForInstanceStateAsync` polls 60× at 5 s (`EC2Helper.cs:131-145`) — a stuck instance gives up after 5 min with no escalation.
- **`describe-environments` is the only env-name validation.** If `App.config:17` says `cross-stitch-com-env-clone` but the live env was renamed, the helper logs "environment was not found" (`ElasticBeanstalkHelper.cs:44-47`) and returns `false`. No exception, no email alert, no halt of the upload flow — the operator must read `txtStatus` to notice.
- **`.ebextensions/05_health_check.config` overrides `01_node.config`.** Both files set `HealthCheckPath`; `05` wins by alphabetical load order. If a future contributor edits `01_node.config:5` expecting it to be authoritative, the change will be silently inert.
- **CloudFront cache lag.** `eb deploy` does not invalidate the CloudFront distribution. Static assets shipped inside the Next.js bundle (under `/_next/...`) are served by the EB instance, not CloudFront, so they update on deploy. But any change touching `/photos/**` or `/pdfs/**` (only the Uploader writes those — see `s3-paths.md` §9) can serve a stale edge object until TTL elapses, and no invalidation step is coded anywhere.
- **EB instance is single-AZ-equivalent.** Even though six subnets are listed (`saved_configs/eb-configuration-2025-12-12.cfg.yml:29, 39`), `t2.small` × 1 means the auto-scaling group is sized 1; any AZ outage affecting that instance takes the site down.
- **Saved-config drift is silent.** Editing in the AWS console without `eb config save` produces no warning anywhere. The 2025-12-12 snapshot is the only on-disk record; see §6 and §11.

## 10. Security & Compliance

- **Secrets-on-operator-disk.** The Uploader reads write-side AWS credentials from the default chain (`%USERPROFILE%\.aws\credentials`, see §5.5). Additionally, `App.config` reads sibling secrets from `App.private.config` (`d:/ann/Git/Uploader/Uploader/App.config:3` — `<appSettings file="App.private.config">`); the private file is gitignored (`d:/ann/Git/Uploader/.gitignore`). Loss of the operator workstation = loss of every write-side credential.
- **Web-app secrets in EB env vars.** The saved snapshot shows `PAYPAL_CLIENT_SECRET: secret` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:25`), which is an obvious placeholder — the live secret is set out-of-band. `ENVIRONMENT-SETUP.md:163-175` advises "Do NOT store secrets in Git" and points at "Elastic Beanstalk environment variables" or "AWS SSM Parameter Store" — but the EB env-vars themselves are visible to anyone with `elasticbeanstalk:DescribeConfigurationSettings` permission on the env, so the boundary is "who has AWS console access", which is the single maintainer.
- **CloudFront serves PDFs and photos publicly.** No signed URLs, no `Origin Access Identity` token surfaces in the URL — the design pipeline is public-by-design (cross-ref `s3-paths.md` §10). Any user who learns an AlbumID + DesignID can construct a URL to the PDF kit.
- **EB instance IAM permissions are not in git.** `aws-elasticbeanstalk-ec2-role` is named (`saved_configs/eb-configuration-2025-12-12.cfg.yml:81`) but its inline/managed policy attachments are not. Audit drift on this role would be invisible to a code reviewer.
- **HTTPS listener policy disagreement.** Snapshot says `ELBSecurityPolicy-2016-08` (`saved_configs/eb-configuration-2025-12-12.cfg.yml:34`); production doc says `ELBSecurityPolicy-TLS13-1-2-2021-06` (`ENVIRONMENT-SETUP.md:69-70`). If the snapshot is authoritative, TLS 1.0/1.1 may still be accepted. SSL Labs grade should be re-checked.
- **HSTS configured at app layer.** `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` per `ENVIRONMENT-SETUP.md:93-94`; sent by Next.js middleware, not the ALB. Bypassed if the listener ever serves HTTP without redirect.
- **SSH disabled by policy, not by code.** `ENVIRONMENT-SETUP.md:112-117, 130-132` documents "no inbound port 22, SSM Session Manager only" — but the instance Security Group is not in git; the policy is enforced via AWS console hygiene.
- **EB env-vars include PAYPAL_CLIENT_ID (a public ID), but the snapshot has the same string in `NEXT_PUBLIC_PAYPAL_CLIENT_ID`** (`:16, :20`) — the duplication is correct (one server-side, one inlined into the client bundle); flagging for clarity.
- **No automated secret rotation.** EB env vars are set once and rotated by hand. PayPal, SES, and AWS credentials all live on this single rotation cadence (i.e., none).
- **Operator workstation is a single point of compromise** for both the Uploader's S3/DDB/SES write credentials and the `eb deploy` permission to mutate production.

## 11. Testing & Validation

Concrete probes (each is a single shell line; expected output is matched against this contract):

- **Confirm live env name matches App.config and EB CLI:**
  ```
  aws elasticbeanstalk describe-environments --application-name cross-stitch-com --region us-east-1 --query "Environments[].{Name:EnvironmentName,Status:Status,Health:Health,Version:VersionLabel,Platform:PlatformArn}" --output table
  ```
  Expected: a row with `Name = cross-stitch-com-env-clone`, `Status = Ready`, `Health ∈ {Green, Yellow}`, `Platform` matching `arn:aws:elasticbeanstalk:us-east-1::platform/Node.js 20 running on 64bit Amazon Linux 2023/...`. If `Name` differs, `Uploader/App.config:17` and `MainWindow.xaml.cs:57` are stale.

- **Diff live config against the 2025-12-12 snapshot** (catches console-side drift):
  ```
  aws elasticbeanstalk describe-configuration-settings --application-name cross-stitch-com --environment-name cross-stitch-com-env-clone --region us-east-1 --output yaml > /tmp/live.yml
  diff -u d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml /tmp/live.yml | head -200
  ```
  Any non-trivial diff is grounds for re-running `eb config save eb-configuration-<today>` and committing.

- **Confirm CloudFront serves the design assets it should:**
  ```
  curl -I https://d2o1uvvg91z7o4.cloudfront.net/photos/<AlbumID>/<DesignID>/4.jpg
  ```
  Expected `HTTP/2 200`; cross-ref the more detailed CloudFront probes in `s3-paths.md` §11.

- **Confirm health endpoint:**
  ```
  curl -I https://cross-stitch.com/api/health
  ```
  Expected `HTTP/2 200`; ALB target-group health-check path (`.ebextensions/05_health_check.config:3`).

- **Confirm EC2 instance is taggable by Uploader:**
  ```
  aws ec2 describe-instances --filters "Name=tag:Name,Values=cross-stitch-com-env-clone" "Name=instance-state-name,Values=running" --region us-east-1 --query "Reservations[].Instances[].InstanceId" --output text
  ```
  Expected: at least one instance ID. If empty, `EC2Helper.RebootInstancesRequest` will log `No instances found for environment 'cross-stitch-com-env-clone'.` (`EC2Helper.cs:31-34`) and return `false`.

- **Confirm code/config name consistency (grep, runnable from either repo root):**
  ```
  rg -n "cross-stitch-com-env-clone" d:/ann/Git/cross-stitch d:/ann/Git/Uploader d:/ann/Git/cross-stitch-platform-docs
  ```
  Expected hits (May 2026): `cross-stitch/.elasticbeanstalk/config.yml:3`, `cross-stitch/ENVIRONMENT-SETUP.md:12`, `Uploader/Uploader/App.config:17`, `Uploader/Uploader/MainWindow.xaml.cs:57`. If the count grows by adding a new file, that file owes a citation here.

- **Confirm `.ebextensions/04_options.config` is in sync with the snapshot:**
  ```
  rg -n "DYNAMODB_TABLE_NAME|AWS_REGION" d:/ann/Git/cross-stitch/.ebextensions d:/ann/Git/cross-stitch/saved_configs
  ```
  Expected: same values in both directories.

- **Validate restart end-to-end (from Uploader):** run with no batch folder selected, click **Restart Elastic Beanstalk** (`MainWindow.xaml:119`). Expected `txtStatus` lines: `Requesting Elastic Beanstalk restart...`, `Restarting Elastic Beanstalk environment 'cross-stitch-com-env-clone' (status: Ready, health: ...).`, `Elastic Beanstalk restart request submitted successfully.`, `Elastic Beanstalk restart requested successfully.` Any other sequence indicates a regression in `ElasticBeanstalkHelper`.

## 12. References

### AWS deployment manifest (cross-stitch repo, the writer)

- `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:1-10` — EB application + branch-default environment.
- `d:/ann/Git/cross-stitch/.ebextensions/01_build.config:1-6` — `npm install` container command.
- `d:/ann/Git/cross-stitch/.ebextensions/01_node.config:1-8` — port `3000`, default health-check path.
- `d:/ann/Git/cross-stitch/.ebextensions/02_swap.config:1-5` — 1 GiB swap.
- `d:/ann/Git/cross-stitch/.ebextensions/04_options.config:1-4` — `AWS_REGION`, `DYNAMODB_TABLE_NAME`.
- `d:/ann/Git/cross-stitch/.ebextensions/05_health_check.config:1-3` — `/api/health` override.
- `d:/ann/Git/cross-stitch/.ebextensions/06_cloudwatch_logs.config:1-40` — CloudWatch agent + log group `/aws/elasticbeanstalk/cross-stitch-env/var/log/myapp/webhook.log`.
- `d:/ann/Git/cross-stitch/.ebextensions/07_ntp.config:1-4` — chronyd.
- `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:1-89` — frozen full snapshot (region, account, platform, VPC, subnets, listener, certs, IAM, env vars).
- `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md:1-176` — production env recreation doc, DNS, TLS, HSTS, SSM, logging, post-deploy checklist.
- `d:/ann/Git/cross-stitch/next.config.js:9-15` — CloudFront `/photos/**` allow-list.

### Uploader-side consumers (the reader/operator)

- `d:/ann/Git/Uploader/Uploader/App.config:4` — `S3BucketName = cross-stitch-designs`.
- `d:/ann/Git/Uploader/Uploader/App.config:17` — `ElasticBeanstalkEnvironmentName = cross-stitch-com-env-clone`.
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:50-51` — `AmazonS3Client` / `AmazonSimpleEmailServiceClient` default ctors (no explicit creds).
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:54-57` — `ElasticBeanstalkHelper` ctor, region `USEast1`, env-name fallback literal.
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:59-60` — `S3Helper` ctor with hardcoded `cross-stitch-designs` bucket and `USEast1` region.
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:366-394` — `BtnRestartEb_Click` (manual restart).
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:664-675` — auto-restart as step 5 of `RunFullUploadFlowAsync`.
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml:119` — XAML `Click="BtnRestartEb_Click"` binding.
- `d:/ann/Git/Uploader/Uploader/Helpers/ElasticBeanstalkHelper.cs:1-74` — `DescribeEnvironmentsAsync` + `RestartAppServerAsync`; error catches.
- `d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs:1-222` — `tag:Name` enumeration, `RebootInstancesAsync`, health-check polling.
- `d:/ann/Git/Uploader/Uploader/Helpers/S3Helper.cs:1-83` — region/bucket wiring for write-side S3 calls (errors swallowed to `Console.WriteLine`).

### Sibling contracts (cross-references, do not duplicate)

- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/dynamodb-schema.md` — `CrossStitchItems`, `CrossStitchUsers`, `PasswordResetTokens`, `SubscriptionEvents` attribute schemas.
- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/s3-paths.md` — bucket layout, photo/PDF/chart key shapes, CloudFront URL templates, CloudFront-as-CDN risk and probes.
- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/pinterest-metadata.md` — Pinterest OAuth + token-file conventions (separate from AWS but shares the operator workstation's credential surface).
- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/album-id.md` and `design-id.md` — entity IDs that flow into DynamoDB and S3 keys.
- `d:/ann/Git/cross-stitch-platform-docs/docs/integration/README.md` — contract folder index and rationale.

### Architecture & policy

- `d:/ann/Git/cross-stitch-platform-docs/CLAUDE.md:42-52` — `do-not-invent` rule that includes "AWS deployment assumptions"; this document closes that item.
- `d:/ann/Git/cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:53, 58, 187-206, 217-220, 263` — AWS facts seed for this contract.
- `d:/ann/Git/cross-stitch-platform-docs/plan/integration-aws/README.md:1-15` — pre-existing proposal-only AWS planning doc (not a contract; replaced as the source of operational truth by this document).
- `d:/ann/Git/cross-stitch-platform-docs/plan/integration/CONTRACT-TEMPLATE.md:1-50` — 12-section template this contract follows.
