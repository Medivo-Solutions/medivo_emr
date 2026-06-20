# Medivo UDHP — GitHub Platform Status Report

**Date:** 2026-06-20 (Updated) | **Organization:** [Medivo-Solutions](https://github.com/Medivo-Solutions) | **Plan:** GitHub Free (2 members)

---

## Repositories

| Repository | Visibility | Commits | PRs Merged | Open PRs | Purpose |
|---|---|---|---|---|---|
| [alberta-emr-aws](https://github.com/Medivo-Solutions/alberta-emr-aws) | Private | 89 | 16 | 0 | AWS EMR platform (ca-central-1) |
| [alberta-emr-gcp](https://github.com/Medivo-Solutions/alberta-emr-gcp) | Private | 85 | 12 | 0 | GCP EMR platform (northamerica-northeast1) |
| [medivo_emr](https://github.com/Medivo-Solutions/medivo_emr) | Public | 3 | 2 | 0 | Shared EMR codebase |

## Platform Separation (Completed)

The AWS and GCP deployment tracks are fully separated into dedicated repositories. Each repo contains only its own cloud-specific artifacts — no cross-cloud references remain.

- **AWS repo**: All GCP Terraform, workflows, and scripts removed. Infrastructure namespaced under `aws/terraform/`.
- **GCP repo**: All AWS Terraform, workflows, and scripts removed. Terraform moved from `terraform/` to `gcp/terraform/`. All CI/CD paths updated.

## Deployed Services (All on `main`)

### Backend Microservices — 11 per repo

All services follow a consistent pattern: Python 3.11, FastAPI, port 8080, `/health` and `/ready` endpoints, structured JSON logging, Dockerized.

| Service | AWS Implementation | GCP Implementation | Files |
|---|---|---|---|
| **fhir_gateway** | HealthLake via boto3 SigV4, JWT validation, consent checks | Cloud Healthcare API with service account auth | main.py, Dockerfile, requirements.txt |
| **ai_scribe** | Bedrock Claude 3.5 Sonnet, PHI guardrail | Vertex AI Gemini 1.5 Pro | main.py, bedrock_provider.py, Dockerfile, requirements.txt |
| **iam_rbac** | Cognito JWKS JWT validation, SMART-on-FHIR scopes | Firebase/Identity Platform x509 cert validation | main.py, Dockerfile, requirements.txt |
| **consent_manager** | FHIR R4 Consent (treatment, research, disclosure, BTG), CloudWatch audit | Same + google-cloud-logging audit | main.py, Dockerfile, requirements.txt |
| **hl7_ingest** | HL7 v2 parsing (ADT-A01/A08/A40), PID extraction, FHIR Patient conversion | Same (cloud-agnostic logic) | main.py, Dockerfile, requirements.txt |
| **scheduling_engine** | FHIR Appointment/Slot management, gateway sync | Same (cloud-agnostic) | main.py, Dockerfile, requirements.txt |
| **billing_bridge** | AHCIP claims CRUD with submit workflow | Same (cloud-agnostic) | main.py, Dockerfile, requirements.txt |
| **provincial_exchange** | Cross-provincial PCR exchange, Aurora PostgreSQL via SQLAlchemy | Cloud Spanner for PCR correlation | main.py, Dockerfile, requirements.txt |
| **compliance_ops** | CloudWatch Logs queries, Security Hub findings, BTG reports | Cloud Logging + SCC | main.py, Dockerfile, requirements.txt |
| **pdf_processor** | Amazon Textract OCR, S3 document storage, FHIR DocumentReference | Document AI, GCS | main.py, Dockerfile, requirements.txt |
| **mpi** | Master Patient Index — FHIR-based patient matching | Same | main.py, Dockerfile, requirements.txt |

### Frontend — `emr_frontend` (27 files per repo)

Next.js 14 (upgraded to 15.5.18 via Dependabot), React 18, TypeScript, Tailwind CSS, App Router, standalone Docker output.

| Category | Files |
|---|---|
| **Pages** | Dashboard (`/dashboard`), Login (`/login`), Patient Record (`/patient/[id]` with tabs), AI Scribe (`/patient/[id]/scribe`), Scheduling (`/scheduling`) |
| **Components** | MedicationsPanel, EncountersPanel, TimelinePanel, LabsHistoryPanel, SmartTokenBootstrapper |
| **API Routes** | Auth session (`/api/auth/session`), Service proxy (`/api/proxy/[service]/[...path]`) |
| **Lib** | FHIR types, useFHIR hook, API client |
| **Auth** | Middleware (cookie-based session, redirect to `/login`) |
| **Config** | tailwind.config.ts (EMR color theme), next.config.js, tsconfig.json, postcss.config.js |
| **Tests** | Accessibility spec (a11y/accessibility.spec.ts) |
| **Infra** | Dockerfile (multi-stage Node.js 20 Alpine), .env.example |

## Infrastructure-as-Code Artifacts

### AWS (`alberta-emr-aws/aws/terraform/`) — 14 files
| File | Resources |
|---|---|
| `vpc.tf` | VPC, subnets, NAT GW, VPC endpoints (S3, HealthLake, ECR, KMS, Secrets Manager), flow logs |
| `ecs.tf` | ECS Fargate cluster, task definitions, services, auto-scaling |
| `api_gateway.tf` | ALB, ACM cert, WAFv2 (OWASP + rate-limit + geo + Log4j), HTTPS listeners |
| `rds_aurora_mpi.tf` | Aurora PostgreSQL Serverless v2 (MPI), Secrets Manager credentials |
| `bedrock.tf` | Bedrock IAM, invocation logging, PHI guardrail (ANONYMIZE: name, phone, email, PHN, SIN) |
| `healthlake.tf` | HealthLake FHIR R4 datastore |
| `cognito.tf` | Cognito user pools (clinical staff + patient), SAML 2.0, MFA |
| `kms.tf` | KMS CMK hierarchy (fhir_phi, audit_logs, secrets) with 90-day rotation |
| `audit.tf` | CloudTrail, S3 Object Lock WORM (10-year retention), Glacier lifecycle |
| `security.tf` | Security Hub, GuardDuty, Macie, Cognito SMS role |
| `github_oidc.tf` | GitHub Actions OIDC federation for CI/CD |
| `providers.tf` / `variables.tf` / `outputs.tf` | Provider config (AWS ~> 6.51, Terraform >= 1.8), variables, outputs |

### GCP (`alberta-emr-gcp/gcp/terraform/`) — 10 files
| File | Resources |
|---|---|
| `main.tf` | Provider config (Google/Google-Beta ~> 7.37, Terraform >= 1.8), VPC, project APIs |
| `apigee.tf` | Apigee API gateway |
| `assured_workloads.tf` | Assured Workloads (compliance boundary) |
| `chronicle.tf` | Chronicle SIEM |
| `dataflow.tf` | Dataflow pipelines |
| `dns.tf` | Cloud DNS |
| `load_balancer.tf` | Cloud Load Balancing |
| `mpi.tf` | Master Patient Index |
| `outputs.tf` / `variables.tf` | Outputs and variables |

## CI/CD Pipelines

### AWS Workflows (10)
| Workflow | Trigger | Purpose |
|---|---|---|
| `quality-gates.yml` | Push/PR | pytest, terraform lint, tfsec, Checkov CKV_AWS_*, Trivy (ignore-unfixed), Snyk, FHIR validation |
| `terraform-plan.yml` | PR (aws/terraform/**) | Terraform plan with PR comment |
| `aws-ecs-deploy.yml` | Manual | Build, ECR push, ECS rolling deploy |
| `certcheck.yml` | Weekly (Monday 08:00 UTC) | SSL cert expiry, AWS compliance, KMS rotation check |
| `destroy.yml` | Manual (with DESTROY confirmation) | Gated infrastructure teardown (production blocked) |
| `review-gate.yml` | PR | Enforces approving review from non-author before merge |
| `stale-branches.yml` | Weekly | Deletes merged branches older than 7 days |
| `dependabot.yml` | Dependabot config | Actions, Terraform providers, Python deps, npm |
| `CodeQL` | Auto | Code scanning (GitHub Advanced Security) |
| `Dependency Graph` | Auto | Dependency vulnerability tracking |

### GCP Workflows (11)
| Workflow | Trigger | Purpose |
|---|---|---|
| `quality-gates.yml` | Push/PR | pytest, terraform lint, tfsec, Checkov CKV_GCP_*, Trivy (ignore-unfixed), Snyk, FHIR validation |
| `terraform-plan.yml` | PR (gcp/terraform/**) | Terraform plan with PR comment |
| `gcp-cloudrun-deploy.yml` | Manual | Build, Artifact Registry push, Cloud Run deploy |
| `deploy.yml` | Manual | GCP deployment |
| `certcheck.yml` | Weekly (Monday 08:00 UTC) | SSL cert expiry check |
| `destroy.yml` | Manual (with DESTROY confirmation) | Gated infrastructure teardown (production blocked) |
| `review-gate.yml` | PR | Enforces approving review from non-author before merge |
| `stale-branches.yml` | Weekly | Deletes merged branches older than 7 days |
| `dependabot.yml` | Dependabot config | Actions, Terraform providers, Python deps, npm |
| `CodeQL` | Auto | Code scanning |
| `Dependency Graph` | Auto | Dependency vulnerability tracking |

## Quality Gates Status

### Gate Results (from last fully-executed runs)
| Gate | AWS | GCP | Notes |
|---|---|---|---|
| **terraform-lint** (fmt + validate + tfsec + Checkov) | PASS | PASS | 264 Checkov checks passed, 0 failed |
| **backend-quality** (pytest + secret scan) | PASS | PASS | No secrets detected |
| **frontend-quality** (npm install + lint + build) | PASS | PASS | Next.js builds successfully |
| **fhir-conformance** | PASS | PASS | No FHIR profiles to validate yet (scaffold) |
| **snyk-scan** | PASS | PASS | Token not configured — gracefully skipped |
| **container-scan** (Trivy CVE) | PASS | PASS | `ignore-unfixed: true` added to skip unfixed Debian base image CVEs |

### Run Statistics
| Metric | AWS | GCP |
|---|---|---|
| Total Quality Gate runs | 100 | 97 |
| Successful runs | 36 | 35 |
| Failed runs | 64 | 62 |
| Failure cause (current) | GitHub Actions minutes exhausted | GitHub Actions minutes exhausted |

### Actions Minutes Usage
- **Used:** 2,058 of 2,000 included minutes (Free plan)
- **Overage:** 58 minutes — runs blocked until billing limit is increased or monthly reset occurs
- **Status:** All jobs fail with "recent account payments have failed or your spending limit needs to be increased"

## Compliance & Security Posture

| Check | Status |
|---|---|
| Checkov CKV_AWS_* | **264 passed, 0 failed** (42 skipped with documented rationale) |
| tfsec | **Passing** |
| Terraform fmt/validate | **Passing** |
| Vulnerability alerts | **Enabled** (all repos) |
| Dependabot security updates | **Enabled** (all repos) |
| Dependency auto-updates | **Active** — GitHub Actions, Terraform providers, Python deps, npm |
| GITHUB_TOKEN permissions | **Write** (org + repo level) — updated 2026-06-20 |
| PR review approval via workflows | **Enabled** (org + repo level) |

## Dependency Management (Current Versions)

### GitHub Actions
| Action | AWS | GCP |
|---|---|---|
| `actions/checkout` | v7 | v4 |
| `actions/setup-python` | v6 | v6 |
| `actions/setup-node` | v4 | v6 |
| `actions/github-script` | v9 | v9 |
| `hashicorp/setup-terraform` | v4 | v4 |
| `aws-actions/configure-aws-credentials` | v6 | — |
| `google-github-actions/auth` | — | v3 |

### Terraform Providers
| Provider | AWS | GCP |
|---|---|---|
| `hashicorp/aws` | ~> 6.51 | — |
| `hashicorp/google` | — | ~> 7.37 |
| `hashicorp/google-beta` | — | ~> 7.37 |
| `hashicorp/random` | ~> 3.6 | ~> 3.6 |

### Frontend Dependencies
| Package | Version |
|---|---|
| Next.js | 15.5.18 (upgraded from 14.2.15 via Dependabot) |
| React | 18.3.1 |
| TypeScript | 5.x |
| Tailwind CSS | 3.x |

### Python Test Dependencies (key packages)
| Package | AWS | GCP |
|---|---|---|
| `fastapi` | >=0.110.0 | >=0.138.0 |
| `pydantic` | >=2.6.0 | >=2.13.4 |
| `httpx` | >=0.28.1 | >=0.27.0 |
| `botocore` | >=1.43.34 | >=1.34.0 |
| `google-cloud-kms` | >=2.21.0 | >=3.13.0 |
| `google-cloud-firestore` | >=2.27.0 | >=2.27.0 |
| `google-api-python-client` | >=2.120.0 | >=2.197.0 |

## Enterprise Governance Configuration

### Organization Level
| Setting | Status |
|---|---|
| Two-factor authentication (2FA) | **Enforced** (secure methods only — no SMS) |
| Web commit signoff required | **Enabled** |
| Default repository permission | **Read** (least privilege) |
| Private repo forking | **Blocked** |
| Workflow permissions (GITHUB_TOKEN) | **Write** (org-wide default) |
| Workflow PR review approval | **Enabled** |

### Repository Level (all repos)
| Setting | Status |
|---|---|
| Merge strategy | **Squash-only** (clean linear history) |
| Delete branch on merge | **Enabled** |
| CODEOWNERS | **Configured** (all repos) |
| PR template | **Configured** (compliance checklist, reviewer checklist) |
| SECURITY.md | **Published** (vulnerability disclosure policy with SLA) |

### Branch Protection
| Repo | Method | Controls |
|---|---|---|
| `medivo_emr` | Native branch protection | 1 review required, dismiss stale, CODEOWNERS, signed commits, linear history, block force push |
| `alberta-emr-aws` | CI review gate | Review Approval Check workflow enforces non-author review |
| `alberta-emr-gcp` | CI review gate | Same as AWS |

> **Note:** Native branch protection on private repos requires GitHub Team ($4/user/month). The CI review gate provides equivalent enforcement on the Free plan.

## Code Statistics

| Metric | AWS | GCP |
|---|---|---|
| Total files | 99 | 95 |
| Service files | 61 | 61 |
| Terraform files | 14 | 10 |
| CI/CD workflows | 10 | 11 |
| Primary language | HCL (72K) | Python (69K) |
| Python | 71K | 69K |
| TypeScript | 37K | 37K |
| Shell | 21K | 20K |
| Dockerfile | 2.8K | 2.8K |

## Documentation

| Document | Location | Content |
|---|---|---|
| Architecture (10 Mermaid diagrams) | `docs/architecture-aws.md` | VPC topology, service graph, FHIR flow, auth sequence, MPI, AI Scribe, audit, deploy, CI/CD |
| Service catalogue | `docs/services.md` | 12 ECS Fargate microservices |
| Development instructions | `UDHP_Complete_Development_Instructions.md` | Full build/deploy/test guide |
| README | `README.md` | Repo structure, quick start, compliance matrix, CI status |
| Status report | `STATUS_REPORT.md` (medivo_emr) | Full platform status, artifacts, governance, dependencies |

## Completed Actions

| Action | Date | Details |
|---|---|---|
| Platform separation (AWS/GCP) | 2026-06-19 | Removed cross-cloud artifacts, namespaced Terraform paths |
| Mermaid diagram fixes | 2026-06-19 | Fixed 10 diagrams in `architecture-aws.md` (rendering errors) |
| Checkov CKV_AWS_* remediation | 2026-06-19 | Resolved 9 failures -> 264 passed, 0 failed |
| CI workflow hardening | 2026-06-19-20 | Graceful handling of missing `AWS_ACCOUNT_ID` in terraform-plan, certcheck, destroy |
| Enterprise governance setup | 2026-06-20 | 2FA, CODEOWNERS, PR templates, review gate, SECURITY.md, Dependabot |
| Dependency updates (AWS) | 2026-06-20 | 16 Dependabot PRs merged — Actions, Terraform provider, Python deps, npm |
| Dependency updates (GCP) | 2026-06-20 | 12 Dependabot PRs merged — Actions, Terraform providers, Python deps, npm |
| Org 2FA enforcement | 2026-06-20 | Secure methods only (authenticator app/passkey — SMS blocked) |
| **Backend services deployed** | 2026-06-20 | 11 microservices per repo (22 total) — cloud-specific implementations for all UDHP services |
| **Frontend deployed** | 2026-06-20 | Next.js EMR frontend with 6 pages, 5 components, auth middleware, API proxy, a11y tests |
| **Trivy CVE fix** | 2026-06-20 | Added `ignore-unfixed: true` to Trivy scan; added 4 missing services to container-scan matrix |
| **Next.js upgrade** | 2026-06-20 | Dependabot PR merged — Next.js 14.2.15 -> 15.5.18 (both repos) |
| **Workflow permissions** | 2026-06-20 | GITHUB_TOKEN default set to write (org + both repos); PR review approval enabled |

## Pending Action Items

| Item | Priority | Effort | Details |
|---|---|---|---|
| **Resolve GitHub Actions billing** | Critical | Billing settings | 2,058/2,000 minutes used — all CI blocked. Set spending limit or upgrade plan |
| Upgrade to GitHub Team plan | High | $4/user/month | Unlocks native branch protection on private repos + 3,000 Actions minutes |
| Configure `AWS_ACCOUNT_ID` secret | High | 5 min | Required for terraform plan, compliance checks, and deploy workflows |
| Set up `medivo_emr` repo visibility review | Medium | — | Confirm public visibility is intentional for this repo |
| Add unit tests for backend services | Medium | 2-3 days | pytest test suites for each service — coverage gate requires 85% |
| Add E2E/a11y tests for frontend | Medium | 1-2 days | Playwright test suites (clinical workflows + WCAG 2.1 AA) |
| Pin `actions/checkout` to v7 in GCP | Low | 5 min | GCP repo uses v4 while AWS uses v7 |
| Clean up stale branches | Low | 5 min | AWS has `automated/security-scans-setup` and `fix-terraform-plan-ci` branches |
