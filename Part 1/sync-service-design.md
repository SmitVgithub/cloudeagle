# sync-service - Deployment & CI/CD Design
 
> 1. **Service:** sync-service (Spring Boot + MongoDB)
> 2. **Cloud:** GCP - asia-south1 (Mumbai)
> 3. **Recommendation:** Cloud Run (Phase 1) → GKE Autopilot (Phase 2)
> 4. **Document version:** 1.0 | May 2026
 
---
 
## Table of Contents
 
1. [Overview](#overview)
2. [Part 1 - CI/CD Pipeline Design](#part-1--cicd-pipeline-design)
---
 
## Overview
 
This document covers the end-to-end deployment strategy for `sync-service`. The infrastructure recommendation is **Cloud Run** for the current startup phase, with a defined migration path to GKE Autopilot as the system grows. MongoDB Atlas (managed) is used instead of self-hosted MongoDB.
 
---
 
## Part 1 - CI/CD Pipeline Design
 
### Branching Strategy
 
```
main ─────────────────────────────────────── production
  └── release/* ───────────────────────────  staging
        └── develop ──────────────────────── qa
              └── feature/* ──────────────── PR only (no deploy)
```
 
| Branch | Environment | Trigger |
|---|---|---|
| `feature/*` | - | Opens a PR against `develop` |
| `develop` | `qa` | Merge triggers QA deploy |
| `release/*` | `staging` | Merge triggers staging deploy |
| `main` | `prod` | Merge + manual approval triggers prod deploy |
 
**Accidental prod deployment prevention** works through two independent gates:
1. Branch protection on `main` - direct pushes are blocked. Every change must come through a reviewed and approved PR.
2. Jenkins `input` step - the pipeline pauses before prod deploy and requires explicit human confirmation with a 30-minute timeout.
---
 
### Semantic Versioning and Docker Image Tagging
 
```
┌──────────────────┬────────────────────────────────────────────────┐
│ Branch           │ Docker Image Tag                               │
├──────────────────┼────────────────────────────────────────────────┤
│ feature/login    │ 1.4.0-feature-login-abc1234                    │
│ develop          │ 1.4.0-qa.14  (build number increments)         │
│ release/1.4.0    │ 1.4.0-rc.3   (release candidate)              │
│ main             │ 1.4.0        (immutable production release)    │
└──────────────────┴────────────────────────────────────────────────┘
```
 
Every image tag is immutable - once pushed to Artifact Registry, a tag is never overwritten. This makes rollback a simple matter of redeploying a prior tag.
 
---
 
### PR Review with OpenAI
 
```
feature/* branch push
        │
        ▼
   Jenkins picks up PR
        │
        ├──────────────────────────────┐
        ▼                              ▼
  SAST Scan                   Post diff to OpenAI
  (OWASP dep-check)           GPT-4o Chat API
  fail on CVSS >= 7                    │
        │                              ▼
        │                     AI review comment
        │                     posted to GitHub PR
        │                     (security, quality,
        │                      logic errors)
        └──────────────────────────────┘
                   │
                   ▼
           Human review required
           (PR approval gate - still mandatory)
                   │
                   ▼
             Merge allowed
```
 
---
 
### Jenkins Pipeline - Full Stage Flow
 
```
╔═════════════════════════════════════════════════════════════════════════╗
║                        ON PULL REQUEST                                  ║
║                                                                         ║
║  Checkout ──► Build & Test ──► SAST Scan ──► OpenAI Review ──► Docker   ║
║                                                              Build      ║
║                                                           (no push)     ║
╚═════════════════════════════════════════════════════════════════════════╝
 
╔═════════════════════════════════════════════════════════════════════════╗
║                      ON MERGE → develop (QA)                            ║
║                                                                         ║
║  Checkout ──► Build ──► Test ──► Docker Build+Push ──► Deploy QA        ║
║                                  (qa tag)              ──► Smoke Test   ║
╚═════════════════════════════════════════════════════════════════════════╝
 
╔═════════════════════════════════════════════════════════════════════════╗
║                   ON MERGE → release/* (Staging)                        ║
║                                                                         ║
║  Checkout ──► Build ──► Test ──► Docker Build+Push ──► Deploy Staging   ║
║                                  (rc tag)              ──► Integration  ║
║                                                             Test        ║
╚═════════════════════════════════════════════════════════════════════════╝
 
╔═════════════════════════════════════════════════════════════════════════════╗
║                      ON MERGE → main (Production)                           ║
║                                                                             ║
║  Checkout ──► Build ──► Test ──► Docker Build+Push ──► Manual Approval      ║
║                                  (release tag)                │             ║
║                                                               ▼             ║
║                                                       Deploy to Prod        ║
║                                                               │             ║
║                                                   ┌───────────┴──────┐      ║
║                                                   ▼                  ▼      ║
║                                             Health Check           FAIL     ║
║                                             PASS (/actuator)                ║
║                                                   │                  │      ║
║                                                   ▼                  ▼      ║
║                                           Notify Slack              Auto    ║
║                                           success ✅              Rollback  ║
║                                                                    to last  ║
║                                                                  stable tag ║
╚═════════════════════════════════════════════════════════════════════════════╝
```
 
---
 
### Rollback Strategy
 
```
Deployment fails (health check timeout)
            │
            ▼
Jenkins reads LAST_STABLE_TAG
(stored after every successful prod deploy)
            │
            ▼
Re-run deployBlueGreen("prod", LAST_STABLE_TAG)
            │
            ▼
Traffic restored to last known-good revision
            │
            ▼
Slack alert fires:
"auto-rolled back to v1.3.0"
```
 
Manual rollback is also available at any time: trigger the `deploy-prod` Jenkins job with any prior image tag as a parameter — no code revert or rebuild needed.
 
---
 
### Deployment Strategy - Blue/Green
 
```
BEFORE DEPLOYMENT
──────────────────────────────────────────
  Load Balancer
       │
       └──── 100% traffic
              │
              ▼
         ┌─────────┐             ┌─────────┐
         │  BLUE   │             │  GREEN  │
         │ v1.3.0  │             │  idle   │
         │  LIVE   │             │(standby)│
         └─────────┘             └─────────┘
 
 
DURING DEPLOYMENT  (new version warming up on green)
──────────────────────────────────────────
  Load Balancer
       │
       └──── 100% traffic (still to blue)
              │
              ▼
         ┌─────────┐             ┌─────────┐
         │  BLUE   │             │  GREEN  │
         │ v1.3.0  │             │ v1.4.0  │◄── health check
         │  LIVE   │             │ warming │    running here
         └─────────┘             └─────────┘
 
 
AFTER CUTOVER  (one atomic traffic switch)
──────────────────────────────────────────
  Load Balancer
       │
       └──────────────────── 100% traffic
                                    │
                                    ▼
         ┌─────────┐             ┌─────────┐
         │  BLUE   │             │  GREEN  │
         │ v1.3.0  │             │ v1.4.0  │
         │(standby)│             │  LIVE   │
         └─────────┘             └─────────┘
              │
              └── instant rollback:
                  flip traffic back to blue
                  (one API call, zero redeployment)
```
 
On Cloud Run this is implemented via named revisions with traffic percentage splits - `gcloud run services update-traffic --to-revisions=GREEN=100`. No load balancer reconfiguration needed.
 
---
 
### Configuration Management
 
```
┌─────────────────────────────────────────────────────────────────┐
│                   Configuration Sources                         │
├──────────────────────────┬──────────────────────────────────────┤
│ Non-secret config         │ Secret config                       │
│                           │                                     │
│ config/                   │ GCP Secret Manager                  │
│  application-qa.yml       │  ├── qa/mongodb-uri                 │
│  application-staging.yml  │  ├── staging/mongodb-uri            │
│  application-prod.yml     │  ├── prod/mongodb-uri               │
│                           │  ├── prod/sendgrid-api-key          │
│ Safe to commit to repo    │  └── prod/jwt-secret                │
│ Injected by Jenkins       │                                     │
│ at deploy time            │ Fetched at startup via              │
│                           │ Spring Cloud GCP Secret Manager     │
│                           │ integration                         │
├──────────────────────────┴──────────────────────────────────────┤
│ Pipeline secrets → GitHub Secrets (never logged)                │
│  ├── GCP_SA_KEY         (deployment service account)            │
│  ├── GITHUB_TOKEN       (PR comment posting)                    │
│  ├── OPENAI_API_KEY     (PR review)                             │
│  └── DOCKER_REGISTRY    (Artifact Registry credentials)         │
└─────────────────────────────────────────────────────────────────┘
```
 
---
