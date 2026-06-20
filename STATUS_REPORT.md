# Medivo UDHP — GitHub Platform Status Report

**Date:** 2026-06-20 (Updated) | **Organization:** [Medivo-Solutions](https://github.com/Medivo-Solutions) | **Plan:** GitHub Free (2 members)

---

## Repositories

| Repository | Visibility | Commits | PRs Merged | Open PRs | Purpose |
|---|---|---|---|---|---|
| [alberta-emr-aws](https://github.com/Medivo-Solutions/alberta-emr-aws) | Private | 30 | 15 | 0 | AWS EMR platform (ca-central-1) |
| [alberta-emr-gcp](https://github.com/Medivo-Solutions/alberta-emr-gcp) | Private | 30 | 12 | 0 | GCP EMR platform (northamerica-northeast1) |
| [medivo_emr](https://github.com/Medivo-Solutions/medivo_emr) | Public | 3 | 2 | 0 | Shared EMR codebase |

## Platform Separation (Completed)

The AWS and GCP deployment tracks are fully separated into dedicated repositories. Each repo contains only its own cloud-specific artifacts — no cross-cloud references remain.

- **AWS repo**: All GCP Terraform, workflows, and scripts removed. Infrastructure namespaced under `aws/terraform/`.
- **GCP repo**: All AWS Terraform, workflows, and scripts removed. Terraform moved from `terraform/` to `gcp/terraform/`. All CI/CD paths updated.

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

### AWS Workflows (7)
| Workflow | Trigger | Purpose |
|---|---|---|
| `quality-gates.yml` | Push/PR | pytest, terraform lint, tfsec, Checkov CKV_AWS_*, Trivy, Snyk, FHIR validation |
| `terraform-plan.yml` | PR (aws/terraform/**) | Terraform plan with PR comment |
| `aws-ecs-deploy.yml` | Manual | Build, ECR push, ECS rolling deploy |
| `certcheck.yml` | Weekly (Monday 08:00 UTC) | SSL cert expiry, AWS compliance, KMS rotation check |
| `destroy.yml` | Manual (with DESTROY confirmation) | Gated infrastructure teardown (production blocked) |
| `review-gate.yml` | PR | Enforces approving review from non-author before merge |
| `stale-branches.yml` | Weekly | Deletes merged branches older than 7 days |

### GCP Workflows (8)
`quality-gates.yml`, `terraform-plan.yml`, `gcp-cloudrun-deploy.yml`, `deploy.yml`, `certcheck.yml`, `destroy.yml`, `review-gate.yml`, `stale-branches.yml`

## Compliance & Security Posture

| Check | Status |
|---|---|
| Checkov CKV_AWS_* | **264 passed, 0 failed** (42 skipped with documented rationale) |
| tfsec | **Passing** |
| Terraform fmt/validate | **Passing** |
| Vulnerability alerts | **Enabled** (all repos) |
| Dependabot security updates | **Enabled** (all repos) |
| Dependency auto-updates | **Active** — GitHub Actions, Terraform providers, Python deps |

## Dependency Management (Current Versions)

All dependencies updated via Dependabot — 27 PRs merged across both repos (zero open).

### GitHub Actions
| Action | AWS | GCP |
|---|---|---|
| `actions/checkout` | v7 | v7 |
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
| Checkov CKV_AWS_* remediation | 2026-06-19 | Resolved 9 failures → 264 passed, 0 failed |
| CI workflow hardening | 2026-06-19–20 | Graceful handling of missing `AWS_ACCOUNT_ID` in terraform-plan, certcheck, destroy |
| Enterprise governance setup | 2026-06-20 | 2FA, CODEOWNERS, PR templates, review gate, SECURITY.md, Dependabot |
| Dependency updates (AWS) | 2026-06-20 | 15 Dependabot PRs merged — Actions, Terraform provider, Python deps |
| Dependency updates (GCP) | 2026-06-20 | 12 Dependabot PRs merged — Actions, Terraform providers, Python deps |
| Org 2FA enforcement | 2026-06-20 | Secure methods only (authenticator app/passkey — SMS blocked) |

## Pending Action Items

| Item | Priority | Effort |
|---|---|---|
| Upgrade to GitHub Team plan | High | $4/user/month — unlocks native branch protection on private repos |
| Configure `AWS_ACCOUNT_ID` secret | High | Required for terraform plan, compliance checks, and deploy workflows |
| Set up `medivo_emr` repo visibility review | Medium | Confirm public visibility is intentional for this repo |
