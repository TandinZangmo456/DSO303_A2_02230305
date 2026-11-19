# Service documentation

Every service in the architecture, why it was chosen over the obvious alternative, how it is configured, and what it connects to.
✅ = also implemented in `infra/template.yaml` · 📐 = design component documented here (not in the template, reason given).

## Edge and access

| Service | Role | Why over an alternative | Key configuration | Connects to |
|---|---|---|---|---|
| Amazon Route 53 📐 | DNS for `exams.<university>` | Alias records to CloudFront are free and health-checked, which a university's external DNS can't do. | Alias A/AAAA → CloudFront, health-checked. *Template uses the default `*.cloudfront.net` name because it needs a registered domain.* | CloudFront |
| Amazon CloudFront ✅ | One HTTPS entry point: caches the frontend, proxies `/api/*` | Serving static files straight from S3 website hosting gives no HTTPS on a custom domain, no WAF and no edge caching. One origin for app + API also means no CORS. | Default behaviour → S3 via **OAC**, `CachingOptimized`. `/api/*` → ALB, `CachingDisabled`, all methods. `redirect-to-https`. | Route 53, WAF, S3 frontend, ALB |
| AWS WAF ✅ (optional flag) | Blocks common web attacks and floods | Security groups only filter IPs and ports. They can't see SQL injection or XSS in HTTP requests. | AWS Managed **CommonRuleSet** + rate-based rule at **20,000 req / 5 min / IP** (kept high because a whole campus can share one NAT IP). `EnableWaf=true`. | CloudFront |
| AWS Certificate Manager 📐 | TLS certificates | Free, auto-renewing. Buying and rotating certs by hand risks an expired cert during an exam week. | Cert in us-east-1 for CloudFront and one for the ALB HTTPS listener, DNS-validated in Route 53. | CloudFront, ALB |
| Amazon Cognito ✅ (optional flag) | Role-based sign-in | Building our own login means storing password hashes and building MFA/reset ourselves. Cognito does it managed and issues JWTs. | User pool, email sign-in, 10-char password policy, groups `students` / `tutors` / `admins`. Access token's `cognito:groups` claim authorises each API call. (`EnableCognito=true` in the template.) | Frontend, API |

## Network

| Service | Role | Why over an alternative | Key configuration | Connects to |
|---|---|---|---|---|
| Amazon VPC ✅ | Private network across 2 AZs | Default VPC has only public subnets, so the DB would get a public route. | `10.0.0.0/16`, 6 subnets (public / app / data × 2 AZs). Data subnets have **no** internet route. | Everything in-VPC |
| Internet Gateway ✅ | Internet access for public subnets | Required for a public ALB and NAT. | Attached to VPC, default route in public route table. | ALB, NAT |
| NAT Gateway ×2 ✅ | Outbound internet for private API instances (package installs) | One shared NAT would be a single point of failure: lose its AZ and the other AZ loses outbound too. A NAT instance needs patching. | One per AZ, each private app route table points to the NAT in its own AZ. | App subnets |
| VPC Gateway Endpoints ✅ | Private path to S3 and DynamoDB | Free (Interface endpoints and NAT are billed per GB). Traffic never leaves the AWS network. | S3 + DynamoDB endpoints on both app route tables. | API, S3, DynamoDB |
| Security groups ✅ | Least-privilege firewalls | Network ACLs are stateless and work on subnets. SGs can reference *each other*, so access follows the tier, not an IP. | ALB :443 ← internet (CloudFront origin-facing prefix list in production). API :8080 ← **ALB SG only**. RDS :3306 ← **API SG only**. | ALB, EC2, RDS |

## Compute

| Service | Role | Why over an alternative | Key configuration | Connects to |
|---|---|---|---|---|
| Application Load Balancer ✅ | Spreads API traffic across both AZs, removes unhealthy targets | Classic/NLB don't do path routing at layer 7. DNS round-robin can't health-check. | Internet-facing in both public subnets, HTTPS :443 listener with ACM cert (HTTP → HTTPS redirect). Target group :8080, health check `/api/health` every 10 s (2 fails = out), deregistration delay 30 s. | CloudFront, EC2 |
| Amazon EC2 + Auto Scaling ✅ | Runs the stateless Flask API | Lambda for the whole API would hit cold starts at the 3,000-student burst and would need VPC networking to reach RDS. A fixed EC2 fleet can't absorb peaks. | Launch template: Amazon Linux 2023, t3.small, **IMDSv2 required**, API instance role. ASG min 2 / max 6 across both app subnets, ELB health checks, CPU target tracking 50 %, **rolling update** when `AppVersion` changes. | ALB, RDS, DynamoDB, S3, SQS, Secrets Manager |
| AWS Lambda ✅ | Async workers: auto-mark MCQs, notify tutors | Running workers on the API instances would compete with exam traffic. Lambda scales per message and costs nothing when idle. | Python 3.12. **auto-mark**: SQS batch 10, max concurrency 5 (protects the table at peak). **notify-tutor**: SQS batch 10. Each has its own execution role. | SQS, DynamoDB, SNS |

## Data and storage

| Service | Role | Why over an alternative | Key configuration | Connects to |
|---|---|---|---|---|
| Amazon RDS for MySQL (Multi-AZ) ✅ | Relational data: exams, attempts, schedules, results, users' roles | MySQL on EC2 means doing replication, failover and backups by hand. Aurora costs more and isn't needed at 500 concurrent users. Multi-AZ gives a **synchronous standby** → automatic failover with no data loss. | db.t3.micro, gp3 20 GB, **MultiAZ: true**, **StorageEncrypted** (KMS), 7-day backups (point-in-time restore = RPO minutes), private data subnets, not public. | API |
| Amazon DynamoDB ✅ | Autosaved answers (~100 writes/s at peak) and marking results | Writing every 30-s autosave into RDS would make the exam peak a DB problem. DynamoDB on-demand absorbs bursts with no capacity planning and is replicated across 3 AZs. | Table `exam-platform-answers`, PK `attemptId` + SK `questionId`, **on-demand**, **PITR on**, SSE on. | API, auto-mark Lambda |
| Amazon S3 (uploads) ✅ | Stores assignment files (PDF/ZIP ≤ 50 MB) | Cheaper and more durable (11 nines) than EBS/EFS, and not tied to one AZ. **Presigned POST** keeps 50 MB uploads off the API. | Versioning, **SSE-KMS**, block all public access, deny non-TLS policy, CORS for POST. Lifecycle: **IA at 30 d → Glacier at 180 d → expire at 1,825 d (5 y)**. Event `ObjectCreated` on `assignments/` → SQS. | Browser (presigned), API, SQS, Lambda, IAM |
| Amazon S3 (frontend) ✅ | Hosts the static web app | A web server on EC2 just for static files is wasted compute. | Private bucket. Only CloudFront can read it (OAC + bucket policy with `AWS:SourceArn`). | CloudFront |
| AWS Backup 📐 | Long-term (5-year) retention of RDS/DynamoDB | RDS automated backups stop at 35 days, but records must be kept for 5 years. | Monthly backup plan, 5-year retention, vault encrypted with KMS. | RDS, DynamoDB |

## Integration and messaging

| Service | Role | Why over an alternative | Key configuration | Connects to |
|---|---|---|---|---|
| Amazon SQS ✅ | Buffers marking jobs and upload events | Calling Lambda/email directly from the request would make students wait, and a failure would lose the job. SQS decouples and retries. Kinesis is overkill for jobs that don't need ordering. | 2 queues, each with a **DLQ (maxReceiveCount 3, 14-day retention)**, SSE-SQS, visibility timeout 60 s (> Lambda timeout). | API, S3, Lambda |
| Amazon SNS ✅ | Tutor email notifications | Lambda calling SMTP directly means managing credentials and retries. SNS fans out to email/SMS/more subscribers. | Topic encrypted with `alias/aws/sns`, email subscription from `TutorEmail`. | Notify Lambda, tutor inbox |
| Amazon SES 📐 | Per-student emails (results released) | SNS email can't send personalised messages to 8,000 different addresses. SES can, with templates and bounce handling. *Not in the template: SES needs a verified domain.* | Verified university domain, DKIM, configuration set for bounces. | Lambda |

## Security, operations and delivery

| Service | Role | Why over an alternative | Key configuration | Connects to |
|---|---|---|---|---|
| AWS Secrets Manager ✅ | DB credentials | Passwords in code, user data or env files end up in Git or AMIs. Secrets Manager is encrypted, audited and can rotate. | Generated 24-char password. RDS reads it via dynamic reference and the API reads it at runtime with its instance role. | RDS, API |
| AWS KMS ✅ | Encryption keys for data at rest | SSE-S3 alone gives no key-usage audit trail. KMS logs every decrypt in CloudTrail. | AWS-managed keys for RDS, S3 (bucket key on), SNS. DynamoDB/SQS SSE on. | RDS, S3, SNS, Secrets Manager |
| AWS IAM ✅ | Permissions for instances and Lambdas | Long-lived access keys on servers can leak. Roles give temporary, auto-rotated credentials. | One role per component: API role (its table, bucket, queues, secret only), one execution role per Lambda. *The template takes the role names as parameters, so it can also run in AWS Academy with `LabRole`.* | EC2, Lambda, S3 |
| Amazon CloudWatch ✅ | Metrics, logs, alarms | Without it you find out about failures from students. | Alarms: ALB 5xx > 20/min, UnHealthyHostCount > 0, DLQ not empty, RDS CPU > 80 %. Lambda logs auto-created. | ALB, RDS, SQS, Lambda |
| AWS CloudTrail 📐 | Audit log of every API call | Needed to prove who changed exams or results. | Trail to an S3 bucket with log-file validation (90-day event history is on by default in every account). | S3 |
| AWS CloudFormation ✅ | Infrastructure as code | Console clicking can't be reviewed or rebuilt within the 1-hour RTO. Terraform would need extra state storage. CFN is native and needs no backend. | `infra/template.yaml`, one stack, parameters for version/WAF/Cognito. | All resources |
| GitHub Actions ✅ | CI/CD: push to `main` redeploys | Sits next to the code, free for this use, and needs no extra pipeline infrastructure. CodePipeline + CodeBuild would work too but adds roles, buckets and cost. | Lint (cfn-lint, pyflakes) → zip API to S3 by commit SHA → `cloudformation deploy` (new SHA = rolling update) → sync frontend + invalidate CloudFront → smoke test. | GitHub, S3, CloudFormation, CloudFront |
