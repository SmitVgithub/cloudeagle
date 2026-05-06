# sync-service - Infrastructure Design
 
> 1. **Service:** sync-service (Spring Boot + MongoDB)
> 2. **Cloud:** GCP - asia-south1 (Mumbai)
> 3. **Recommendation:** Cloud Run (Phase 1) → GKE Autopilot (Phase 2)
> 4. **Document version:** 1.0 | May 2026
 
---
 
## Table of Contents
 
1. [Part 2 - Infrastructure Design](#part-2--infrastructure-design)
2. [Migration Path](#migration-path)
---

## Part 2 - Infrastructure Design
 
### Full Architecture - Request Flow Diagram
 
```
                     ┌─────────────────────────────────────────────────────────┐
                     │                         GCP                             │
  ┌────────────┐     │  ┌─────────────────────────────────────────────────┐    │
  │            │     │  │                   VPC                           │    │
  │ Customers  │     │  │  ┌──────────────────────────────────────────┐   │    │
  │ (browser + │     │  │  │         Region: asia-south1 (Mumbai)     │   │    │
  │  mobile)   │     │  │  │                                          │   │    │
  └─────┬──────┘     │  │  │  ┌───────────────────────────────────┐   │   │    │
        │            │  │  │  │      Cloud Run Services           │   │   │    │
        │ PORT 443   │  │  │  │                                   │   │   │    │
┌───────┘ HTTPS      │  │  │  │ ┌──────────┐  ┌──────────────┐    │   │   │    │
│                    │  │  │  │ │  QA      │  │   Staging    │    │   │   │    │
│ ┌───────────────┐  │  │  │  │ │ min:0    │  │   min:1      │    │   │   │    │
│ │  Cloud Armor  │  │  │  │  │ │ max:5    │  │   max:8      │    │   │   │    │
│ │  (WAF + DDoS) │  │  │  │  │ └──────────┘  └──────────────┘    │   │   │    │
│ └───────┬───────┘  │  │  │  │                                   │   │   │    │
│         │          │  │  │  │ ┌─────────────────────────────┐   │   │   │    │
└───┐     │ HTTP :80 │  │  │  │ │   Production                │   │   │   │    │
    ▼     ▼          │  │  │  │ │   Blue  v1.3.0 →   0%       │   │   │   │    │
  ┌───────────────┐  │  │  │  │ │   Green v1.4.0 → 100% ✓     │   │   │   │    │
  │  HTTPS        │  │  │  │  │ │   min:1  max:20             │   │   │   │    │
  │  Load Balancer│──┼──┼──┼──┼►│   SA: sync-svc-prod@...     │   │   │   │    │
  │  (managed SSL)│  │  │  │  │ └──────────────┬──────────────┘   │   │   │    │
  └───────────────┘  │  │  │  │                │                  │   │   │    │
          │          │  │  │  │  Serverless VPC │ Connector       │   │   │    │
          │          │  │  │  │  (10.8.0.0/28)  │                 │   │   │    │
          │          │  │  │  └───────────────────────────────────┘   │   │    │
          │          │  │  │                   │                      │   │    │
          │          │  │  │    ┌──────────────┼──────────────────┐   │   │    │
          │          │  │  │    │ Data Layer   │                  │   │   │    │
          │          │  │  │    │              ▼                  │   │   │    │
          │          │  │  │    │  ┌──────────────────┐           │   │   │    │
          │          │  │  │    │  │  MongoDB Atlas   │           │   │   │    │
          │          │  │  │    │  │  M10 Dedicated   │           │   │   │    │
          │          │  │  │    │  │  Port 27017 TCP  │           │   │   │    │
          │          │  │  │    │  │  VPC Peering     │           │   │   │    │
          │          │  │  │    │  │  3-node replica  │           │   │   │    │
          │          │  │  │    │  └──────────────────┘           │   │   │    │
          │          │  │  │    │                                 │   │   │    │
          │          │  │  │    │  ┌──────────────────┐           │   │   │    │
          │          │  │  │    │  │  Redis           │           │   │   │    │
          │          │  │  │    │  │  Memorystore 1GB │           │   │   │    │
          │          │  │  │    │  │  Port 6379 TCP   │           │   │   │    │
          │          │  │  │    │  │  HA replica      │           │   │   │    │
          │          │  │  │    │  └──────────────────┘           │   │   │    │
          │          │  │  │    │                                 │   │   │    │
          │          │  │  │    │  ┌──────────────────┐           │   │   │    │
          │          │  │  │    │  │  Cloud Storage   │           │   │   │    │
          │          │  │  │    │  │  Standard tier   │           │   │   │    │
          │          │  │  │    │  │  HTTPS (priv SC) │           │   │   │    │
          │          │  │  │    │  └──────────────────┘           │   │   │    │
          │          │  │  │    └─────────────────────────────────┘   │   │    │
          │          │  │  │                                          │   │    │
          │          │  │  │  ┌────────────────────────────────────┐  │   │    │
          │          │  │  │  │  Cloud NAT (asia-south1)           │  │   │    │
          │          │  │  │  │  Outbound egress, no public IPs    │  │   │    │
          │          │  │  │  └──────────────────────┬─────────────┘  │   │    │
          │          │  │  └─────────────────────────┼────────────────┘   │    │
          │          │  └────────────────────────────┼────────────────────┘    │
          │          └───────────────────────────────┼─────────────────────────┘ 
          │                                          │                          
  ┌───────▼──────┐                                   │                          
  │   Internet   │◄──────────────────────────────────┘                          
  │  (external   │  External APIs, SendGrid (HTTPS :443 via Cloud NAT)         
  │   services)  │                                                             
  └──────────────┘                                                             

```
 
---
 
### Supporting Services - Right Panel
 
```
┌───────────────────────────────────────────────────────────────────┐
│                       Other GCP Resources                         │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Secret Manager                          ◄── TCP from Cloud Run   │
│  ├── Env-scoped secrets (qa/stg/prod)                             │
│  ├── secretmanager.accessor IAM role                              │
│  └── No cross-env secret access                                   │
│                                                                   │
│  Artifact Registry                       ◄── push from Jenkins    │
│  ├── gcr.io / asia-south1                                         │
│  ├── Immutable semantic tags                                      │
│  └── Vulnerability scanning enabled                               │
│                                                                   │
│  Cloud Monitoring                        ◄── TCP from Cloud Run   │
│  ├── Latency: P50 / P95 / P99 dashboard                           │
│  ├── Error rate alerts (>1% over 5min)                            │
│  └── Cloud Trace auto-instrumented                                │
│                                                                   │
│  Cloud Logging                           ◄── stdout/stderr auto   │
│  ├── Structured JSON log collection                               │
│  ├── Log-based metrics                                            │
│  └── Export to BigQuery (retention)                               │
│                                                                   │
│  Cloud IAM                               ◄── TCP all services     │
│  ├── Per-env service accounts                                     │
│  ├── Least privilege enforced                                     │
│  └── No key files distributed                                     │
│                                                                   │
│  Security Command Center                 ◄── TCP from platform    │
│  ├── Threat detection                                             │
│  ├── IAM anomaly alerts                                           │
│  └── Vulnerability findings                                       │
│                                                                   │
│  SendGrid (external)                     ◄── via Cloud NAT egress │
│  ├── Transactional email                                          │
│  ├── API key stored in Secret Manager                             │
│  └── HTTPS :443 outbound                                          │
│                                                                   │
│  Cloud Billing                                                    │
│  ├── Budget alerts configured                                     │
│  └── Cost export to BigQuery                                      │
│                                                                   │
│  Jenkins CI/CD                           ──► deploys to Cloud Run │
│  ├── gcloud run deploy via SA key                                 │
│  ├── GitHub Secrets → pipeline secrets                            │
│  └── Pushes images to Artifact Registry                           │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```
 
---
 
### MongoDB Atlas - Why Not Self-Hosted
 
```
Self-hosted MongoDB on Compute Engine
──────────────────────────────────────
Things your team now owns:
  ├── Replica set configuration and failover
  ├── Backup scheduling and restoration testing
  ├── Version upgrades (with planned downtime)
  ├── Monitoring: replication lag, oplog size, connection pool
  ├── Disk provisioning and IOPS tuning
  ├── OS-level security patching
  └── 2am incident response when a node fails
 
MongoDB Atlas M10  (~$57/month)
────────────────────────────────
What Atlas handles for you:
  ├── 3-node replica set, auto-configured
  ├── Daily automated backups, point-in-time restore
  ├── One-click version upgrades (zero-downtime rolling)
  ├── Built-in performance monitoring dashboard
  ├── Automatic disk scaling
  ├── Security patching at the database layer
  └── VPC Peering keeps all traffic off the public internet
 
For a startup: Atlas pays for itself in engineering time saved
within the first month. Self-hosting is only worth reconsidering
if data residency law explicitly requires it.
```
 
---
 
### Compute Options Comparison
 
```
┌─────────────────┬──────────────────┬──────────────────┬──────────────────┐
│                 │  Cloud Run ✓     │  GKE Autopilot   │ Compute Engine   │
│                 │  RECOMMENDED     │  (Phase 2)       │ (special cases)  │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Cluster mgmt    │ None             │ Minimal          │ Manual (MIGs)    │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Cold start      │ 2–5s (mitigated  │ ~1s              │ ~30–90s (VM)     │
│                 │ with min=1)      │                  │                  │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Autoscaling     │ Request-based    │ CPU / Custom HPA │ CPU (MIG)        │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Blue/Green      │ Native revision  │ Two Deployments  │ Two MIGs         │
│                 │ traffic split    │ + svc selector   │ + LB label swap  │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Scale to zero   │ Yes (QA: free)   │ No               │ No               │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Cost (all envs) │ ~$75–100/month   │ ~$230–350/month  │ ~$120–200/month  │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Best for        │ Startup, low ops │ Multi-service,   │ GPU, local disk, │
│                 │ fast ship        │ complex platform │ on-prem migration│
└─────────────────┴──────────────────┴──────────────────┴──────────────────┘
```
 
---
 
### Networking Architecture
 
```
Internet
    │
    │ HTTPS :443
    ▼
┌────────────────────────────────────────┐
│  Cloud Armor                           │
│  WAF rules, rate limiting,             │
│  DDoS protection, OWASP top-10 preset  │
└────────────────┬───────────────────────┘
                 │ HTTP :80 (internal)
                 ▼
┌────────────────────────────────────────┐
│  HTTPS Load Balancer                   │
│  Google-managed SSL certificate        │
│  Health checks on /actuator/health     │
│  Serverless NEG backend                │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Serverless VPC Access Connector       │
│  Subnet: 10.8.0.0/28                   │
│  Region: asia-south1                   │
│  Bridges Cloud Run into private VPC    │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│  Cloud Run (private ingress)           │
│  No public IP on instances             │
│  All traffic routed through LB only    │
└───────────┬─────────────┬──────────────┘
            │             │
      :27017 TCP      :6379 TCP
      VPC Peer         Private
            │             │
            ▼             ▼
     MongoDB Atlas    Redis Memorystore
 
Outbound (to external APIs, SendGrid):
  Cloud Run → Cloud NAT → Internet
  No public IPs on Cloud Run instances
 
Firewall rules (least privilege):
  Allow :443   from 0.0.0.0/0   → Load Balancer only
  Allow :80    from LB ranges   → VPC Connector
  Allow :27017 from VPC subnet  → MongoDB Atlas
  Allow :6379  from VPC subnet  → Redis
  Deny  all other inbound
```
 
---
 
### Secrets and IAM Architecture
 
```
┌──────────────────────────────────────────────────────────┐
│                 Secrets Architecture                     │
│                                                          │
│  GitHub Secrets  (pipeline only, never logged)           │
│  ├── GCP_SA_KEY        → Jenkins deploys to Cloud Run    │
│  ├── GITHUB_TOKEN      → Posts PR review comments        │
│  ├── OPENAI_API_KEY    → PR review via GPT-4o            │
│  └── DOCKER_REGISTRY   → Pushes to Artifact Registry     │
│                                                          │
│  GCP Secret Manager  (runtime, env-scoped)               │
│  ├── qa/mongodb-uri       → QA service account only      │
│  ├── staging/mongodb-uri  → Staging SA only              │
│  ├── prod/mongodb-uri     → Prod SA only                 │
│  ├── prod/sendgrid-key    → Prod SA only                 │
│  └── prod/jwt-secret      → Prod SA only                 │
│                                                          │
│  IAM Bindings  (least privilege)                         │
│  ├── sync-svc-qa@...   secretmanager.accessor (qa/*)     │
│  ├── sync-svc-stg@...  secretmanager.accessor (stg/*)    │
│  └── sync-svc-prod@... secretmanager.accessor (prod/*)   │
│                                                          │
│  No service account key files distributed anywhere.      │
│  Cloud Run identity set declaratively at deploy time.    │
│  Spring Cloud GCP fetches secrets at app startup.        │
└──────────────────────────────────────────────────────────┘
```
 
---
 
### Logging and Monitoring
 
```
Cloud Run Instance
    │ stdout/stderr  (structured JSON logs)
    ▼
Cloud Logging
  ├── Log Explorer   → ad-hoc queries by field
  ├── Log-based metrics  → custom counters from log patterns
  └── BigQuery export    → long-term retention + analytics
    │
    ▼
Cloud Monitoring
  ├── Dashboard: P50 / P95 / P99 latency, error rate, instance count
  ├── Alert: error rate > 1%    over 5 min  → Slack / PagerDuty
  ├── Alert: P99 latency > 2s   over 5 min  → Slack / PagerDuty
  └── Alert: Cloud Run instance restart       → Slack
    │
Cloud Trace
  ├── Auto-instrumented via Spring Cloud GCP Trace
  ├── Trace context propagated into MongoDB + Redis calls
  └── Full latency breakdown per request in GCP Console
 
MongoDB Atlas Monitoring (separate)
  ├── Connection pool saturation alerts
  ├── Replication lag monitoring
  └── Slow query log (Atlas console + alerts)
```
 
---
 
## Migration Path
 
```
Phase 1  (Now — startup)            Phase 2  (6–18 months — growth)
─────────────────────────           ────────────────────────────────
Cloud Run                           GKE Autopilot
+ MongoDB Atlas          ────►      + MongoDB Atlas
+ Redis Memorystore                 + Redis Memorystore
+ Cloud Storage                     + Cloud Storage
~$75–100/month                      ~$230–350/month
 
What stays exactly the same:        What changes:
  ✓ Docker image format               Cloud Run → Kubernetes Deployment
  ✓ Semantic versioning scheme        Service Account → Workload Identity
  ✓ Jenkinsfile (minor gcloud tweak)  Traffic split → K8s svc selector
  ✓ GCP Secret Manager config         Serverless VPC → node VPC native
  ✓ MongoDB Atlas setup
  ✓ Cloud Logging / Monitoring
  ✓ Cloud Armor + Load Balancer
  ✓ Artifact Registry
 
When to trigger migration:
  ├── More than 15 microservices needing inter-service networking
  ├── Custom metric HPA needed (beyond request count / CPU)
  ├── Service mesh or advanced canary routing required
  └── Team has invested time in Kubernetes expertise
```
 
---
