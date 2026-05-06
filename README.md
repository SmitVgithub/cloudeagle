# CloudEagle - DevOps Technical Assessment

This repository contains the complete CI/CD pipeline design and infrastructure architecture for **sync-service**, a Spring Boot application with MongoDB, deployed on Google Cloud Platform.

---

## Overview

This assessment demonstrates:
- Production-ready CI/CD pipeline with Jenkins
- Blue/Green deployment strategy for zero-downtime releases
- Cloud Run infrastructure architecture (recommended for startup phase)
- Automated rollback on deployment failure
- AI-powered PR reviews via OpenAI
- Multi-environment configuration management (QA, Staging, Production)

---

## Repository Structure

```
.
├── README.md                                  # This file
├── Part-1/                                    # CI/CD Pipeline Design
│   ├── sync-service-design.md                 # Pipeline architecture document
│   ├── Jenkinsfile                            # Production pipeline implementation
│   └── ai_review.png                          # PR review by AI
│
└── Part-2/                                    # Infrastructure Design
    ├── sync-service-infrastructure-design.md  # Architecture document
    └── Cloudeagle_Architecture_Diagram.png    # Supporting architecture visuals
```

---

## Part 1 - CI/CD Pipeline Design

**Location:** `Part-1/`

### What's Covered:

- **Branching Strategy**: Feature → Develop → Release → Main with environment mapping
- **Semantic Versioning**: Automatic image tagging based on branch context
- **PR Review Automation**: OpenAI GPT-4o reviews code diffs for security and quality issues
- **Jenkins Pipeline**: Multi-stage pipeline with SAST, testing, Docker build, and deployment
- **Blue/Green Deployment**: Zero-downtime releases with atomic traffic cutover
- **Rollback Strategy**: Automatic rollback on health check failure
- **Configuration Management**: Environment-specific configs and secret handling via GCP Secret Manager

### Key Files:

- `sync-service-design.md` - Full pipeline architecture documentation
- `Jenkinsfile` - Production-ready declarative pipeline with:
  - Semantic version resolution from `VERSION` file
  - OWASP dependency check (SAST)
  - OpenAI-powered PR review
  - Docker multi-stage build
  - Blue/Green deployment function
  - Auto-rollback on failure
  - Slack notifications

---

## Part 2 - Infrastructure Design

**Location:** `Part-2/`

### What's Covered:

- **Recommended Architecture**: Cloud Run on GCP (Mumbai region)
- **Compute Options Analysis**: Cloud Run vs GKE Autopilot vs Compute Engine
- **MongoDB Hosting**: MongoDB Atlas (managed) - not self-hosted
- **Networking**: VPC, Serverless VPC Connector, Cloud Armor, HTTPS Load Balancer, Cloud NAT
- **Security**: IAM service accounts, GCP Secret Manager, least privilege access
- **Observability**: Cloud Logging, Cloud Monitoring, Cloud Trace
- **Migration Path**: Cloud Run (Phase 1) → GKE Autopilot (Phase 2)

### Key Files:

- `sync-service-infrastructure-design.md` - Full infrastructure documentation with ASCII diagrams
- `Cloudeagle_Architecture_Diagram.png` - Detailed interactive diagram showing:
  - Full request flow from customer to database
  - All GCP services with ports and protocols
  - Blue/Green deployment topology
  - Supporting services (Secret Manager, Artifact Registry, IAM, etc.)

---

## Quick Start

### View the Architecture Diagram

Open `Part-2/Cloudeagle_Architecture_Diagram.png`.

### Deploy via Jenkins

1. Set up Jenkins with the required credentials:
   - `GCP_SA_KEY` - GCP service account key for deployment
   - `GITHUB_TOKEN` - GitHub token for PR comments
   - `OPENAI_API_KEY` - OpenAI API key for PR reviews

2. Update `Jenkinsfile` line 9:
   ```groovy
   GCR_REGISTRY = "gcr.io/sync-service-registry"
   ```

3. Commit to a feature branch and open a PR to trigger the pipeline.

---

## Infrastructure Recommendation

**For startup phase (now):**
- **Cloud Run** with min-instances=1 on production
- **MongoDB Atlas M10** (managed, not self-hosted)
- **Total cost:** ~$75–100/month

**For growth phase (6–18 months):**
- **GKE Autopilot** for multi-service orchestration
- Same MongoDB Atlas setup
- **Total cost:** ~$230–350/month

---

## Key Technical Decisions

### Why Cloud Run over GKE (for now)?
- Zero cluster management overhead
- Native Blue/Green via traffic splitting
- Scale to zero on QA environment
- Lowest cost for startup constraints
- Clean migration path to GKE when needed

### Why MongoDB Atlas over self-hosted?
- Automated backups and point-in-time restore
- Zero-downtime version upgrades
- Built-in monitoring and alerting
- VPC Peering for private connectivity
- Saves significant engineering time

### Why Blue/Green over Rolling?
- Atomic traffic cutover (zero downtime)
- Instant rollback capability
- No version mixing during deployment
- Critical for Spring Boot + MongoDB compatibility

---

## Cost Breakdown (Cloud Run Architecture)

| Service | Monthly Cost |
|---------|-------------|
| Cloud Run (prod min=1, staging min=1, QA min=0) | $20–35 |
| MongoDB Atlas M10 Dedicated | $57 |
| Cloud Storage (Standard tier) | $5–10 |
| Load Balancer + Cloud Armor | $20–25 |
| Artifact Registry + other services | $5–10 |
| **Total** | **~$75–100** |

---

## Security Highlights

- **No public IPs** on Cloud Run instances (private ingress only)
- **GCP Secret Manager** for all runtime secrets (env-scoped IAM)
- **Cloud Armor** WAF with OWASP top-10 protection
- **VPC Peering** to MongoDB Atlas (traffic never hits public internet)
- **IAM least privilege** — separate service accounts per environment
- **No service account key files** distributed (Cloud Run identity set at deploy time)

---

## CI/CD Pipeline Flow

```
feature/* → PR opened → Jenkins triggers:
  ├── Build & Unit Test
  ├── SAST Scan (OWASP dependency check)
  ├── OpenAI PR Review (security, quality, logic)
  └── Docker Build (no push)

develop → merge → Deploy to QA
  └── Smoke test

release/* → merge → Deploy to Staging
  └── Integration test

main → merge → Manual Approval (30min timeout)
  └── Deploy to Production (Blue/Green)
      ├── Health check PASS → Notify Slack ✅
      └── Health check FAIL → Auto-rollback to last stable tag
```

---

## Technologies Used

- **Backend**: Spring Boot (Java)
- **Database**: MongoDB Atlas (managed)
- **Compute**: Google Cloud Run
- **CI/CD**: Jenkins (declarative pipeline)
- **Container Registry**: Google Artifact Registry
- **Secrets**: GCP Secret Manager
- **Monitoring**: Cloud Logging, Cloud Monitoring, Cloud Trace
- **Security**: Cloud Armor, Cloud IAM, Security Command Center
- **AI**: OpenAI GPT-4o (PR review automation)

---

## Assessment Requirements Met

**Part 1 - CI/CD Design**
- Branching strategy with environment mapping
- Accidental prod deployment prevention (2 gates)
- Jenkins pipeline with all stages documented
- Rollback strategy (automatic + manual)
- Configuration management (Secret Manager + GitHub Secrets)
- Blue/Green deployment justification

**Part 2 - Infrastructure Design**
- Cloud Run recommended with justification
- Alternative compute options evaluated (GKE, Compute Engine)
- MongoDB Atlas vs self-hosted comparison
- VPC networking with Cloud NAT and private ingress
- IAM and secrets architecture
- Logging and monitoring stack
- Architecture diagram with request flow

---

## Author

DevOps Technical Assessment - May 2026

---
