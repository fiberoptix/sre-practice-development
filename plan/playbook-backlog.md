# SRE Playbook — Backlog of Topics to Flesh Out

> These are notes from the strategic planning session. Each item needs
> a detailed, step-by-step section in the full SRE Playbook.
> Items marked [DISCUSSED IN DETAIL] had significant conversation and
> have a clear direction. Items marked [NEEDS DISCUSSION] are in the
> strategic plan but haven't been explored yet.

## Key Context (for anyone reading this later)

### Infrastructure ownership model
- **Middle Office controls:** Docker containers, Dockerfiles, GitLab CI
  pipelines, Artifactory images, assembly files, Nomad/AppEngine deployments
- **Enterprise teams control:** VMs (VMWare), Linux, databases (Oracle,
  Postgres), message queues (IBM MQ, RabbitMQ), the Nomad/AppEngine
  cluster infrastructure itself
- Enterprise infra provisioning takes 6–9 months via SNOW requests
- Hardware is paid for up front via internal chargeback model

### Build/deploy flow
GitLab CI builds Docker image → pushes to Artifactory → bash scripts
generate assembly files at build time from version-controlled indexes
+ resource configs → assemblies stored in Artifactory → pipeline applies
assembly to Nomad/AppEngine during change release. DEV teams own the
assembly generation; SRE owns the pipeline standards that wrap around it.

### Current release cadence
- ~50 releases per month
- BNY enterprise pipeline standard already adopted
- High rate of eRFCs (emergency changes) and LTEs (late-to-execute)
- Most Monday incidents trace back to weekend changes + human error + lack of QA

### Monitoring stack
Splunk, AppDynamics, Grafana, Moogsoft, MyCIO — strong operational
monitoring but fragmented across tools with no unified SLI/SLO/error
budget reliability view.

---

## 1. Base Image Compliance System [DISCUSSED IN DETAIL]

Two-pronged enforcement strategy to replace the policy-only approach
that achieved 80% vuln reduction then decayed.

### Prong 1: Build-Breaker
- GitLab CI stage that parses `FROM` in Dockerfiles
- Checks against a maintained whitelist of approved base images
- Rejects builds on non-whitelisted or >6-month-old images
- Whitelist published quarterly aligned with base image release cadence
- Needs: whitelist format, storage location, update process, exception/waiver workflow

### Prong 2: Scheduled Refresh Pipeline (detailed design trimmed from strategic plan)
- Monthly/quarterly GitLab scheduled job iterates every container in the service catalog
- For each: checks out repo → swaps `FROM` to latest approved base → rebuilds → runs existing test suite
- **If tests pass:** Opens a merge request for dev team to review and merge (minimal effort)
- **If tests fail:** Creates JIRA assigned to owning team with link to CI output and remediation deadline
- **If no test suite exists:** Higher-priority JIRA — service is both untested AND on a stale base
- Compliance dashboard in MyCIO/Grafana: % current, % with open MRs, % with open JIRAs
- Needs: service catalog/registry as source of truth, repo-to-team ownership mapping, JIRA automation, MR template, deadline policy

### Key design question
- How to handle base image swap in Dockerfiles that use multi-stage builds or parameterized FROM (ARG + FROM)
- Exception/waiver process for services with legitimate pinning requirements

---

## 2. SRE Dashboard [DISCUSSED IN DETAIL]

MyCIO shows what's broken NOW. The SRE Dashboard shows reliability
OVER TIME — SLIs, SLO compliance, error budget burn.

### Build Options Discussed (trimmed from strategic plan, needs full design)
- **Option A — Grafana:** Splunk + AppDynamics data source plugins, native SLO plugin, PromQL, burn rate alerts, multi-dimensional SLOs. Open-source, free, SRE industry standard.
- **Option B — Extend MyCIO:** Zero adoption friction (teams already use it daily). Requires custom dev for SLO math, burn rate, alerting.
- **Recommended — Both:** Grafana as SRE engineering layer. MyCIO as leadership/on-call consumption layer. Grafana feeds MyCIO or both query same telemetry.
- Needs: Grafana deployment architecture, data source plugin configuration, SLO definition workshop output, dashboard mockups, MyCIO integration plan
- Platform decision to be made in Phase 0 during stakeholder alignment

### Why Grafana keeps coming up
- Not a monitoring tool — it's a visualization/query layer over existing tools
- Has official plugins for Splunk and AppDynamics (no data migration)
- Native SLO plugin: define SLI + target → auto-calculates error budget, burn rate, generates alerts
- Supports multi-dimensional SLOs (per region, per client, per service)
- Open-source — can deploy on a VM without procurement
- Industry standard: Vanguard, DBS, Standard Chartered all converged on it

---

## 3. Incident Severity Classification Problem [DISCUSSED IN DETAIL]

People downgrade incidents to P3/P4 to avoid executive visibility.
Most P3s flood in on Mondays after weekend changes.

### How SLOs solve this
- When SLIs measure real metrics against SLO targets, the numbers declare severity
- Error budget burn doesn't care what someone labeled the ticket
- If 0.3% of settlements failed and SLO is 99.9%, the dashboard shows budget is blown
- Removes the human judgment / political incentive to downgrade
- Needs: SLO-to-severity mapping policy, automated alerting thresholds tied to SLIs not manual labels

---

## 4. Remediation Pipeline (RCA → Engineering Action) [DISCUSSED IN DETAIL]

RCAs happen fast (same day). PRBs get exec briefings. But findings
die there — no systematic bridge to engineering work.

### What to build
- Mandatory "remediation actions" section on every PRB
- Each action classified: new alert, monitoring gap, QA test, code fix, policy change, runbook update
- Actions routed into JIRA with SRE/Dev ownership and due dates
- Monthly review: which remediations shipped, which incidents would have been prevented
- Preventive controls catalog: what monitoring/alerting/QA should have caught this?
- Needs: PRB template update in ServiceNow, JIRA automation rules, classification taxonomy, monthly review cadence/format

---

## 5. SLI / SLO / Error Budget Framework [NEEDS DISCUSSION]

- Which services get SLOs first (top-5 by incident volume?)
- What SLIs to measure per service type (trade processing latency, settlement success rate, message throughput, etc.)
- SLO target-setting workshops with business stakeholders
- Error budget policy: what happens when budget is exhausted (release freeze? reliability sprint?)
- Burn rate alerting thresholds (fast-burn vs slow-burn)
- Weekly SLO burn report format and audience
- Needs: service catalog, SLI instrumentation plan per service, stakeholder workshop format

---

## 6. Toil Measurement and Reduction [NEEDS DISCUSSION]

- 2-week time study methodology: how to classify toil vs. engineering
- Toil taxonomy: categories, tracking format, reporting
- Ranking formula: frequency × time × risk of error
- Quarterly toil reduction targets
- Needs: time tracking tool/method, classification criteria, baseline measurement plan

---

## 7. CI/CD Maturity — Beyond Base Images [NEEDS DISCUSSION]

- Canary / blue-green deployment patterns for Middle Office services
- Automated post-deploy smoke tests
- Error-budget-aware release gating (block deploys when budget exhausted)
- Deploy-time metric capture (before/after latency, error rate)
- Change failure rate tracking per service
- **eRFC and LTE reduction** — currently high rate of emergency changes
  and late-to-execute releases across ~50 releases/month. Target: −50%.
  Better pipeline quality gates should reduce both.
- Needs: current pipeline audit, deployment pattern assessment per service, rollback automation design, eRFC/LTE baseline measurement

---

## 8. Pipeline Standardization [DISCUSSED IN DETAIL]

DEV teams own the build/assembly/deploy process (Dockerfiles, bash-generated
assemblies from GitLab-controlled indexes + resource configs → Artifactory → Nomad).
SRE's role is standardizing the CI/CD pipeline templates across services.

### What SRE controls
- GitLab CI pipeline templates with built-in quality gates
- Base image compliance enforcement (build-breaker + scheduled refresh)
- Post-deploy validation stages
- Standard linting, testing, and security scanning stages

### What DEV teams own
- Dockerfiles, assembly generation scripts, resource configs
- Application-level tests
- Merge and promotion through their release process

### DevOps Deep-Dive (own playbook section — two distinct efforts)

**Effort 1: Extend the Middle Office Pipeline Standard**
BNY enterprise pipeline standard is already adopted. The work is
designing and engineering MO-specific extensions on top of it:
- Blue-green deployment patterns for Nomad/AppEngine services
- Canary deployments with automated rollback
- Post-deploy validation stages and automated smoke tests
- Error-budget-aware release gating
- How these patterns work within the existing GitLab → Artifactory → Nomad flow
- Needs: design docs, pilot with one service, stakeholder buy-in from enterprise DevOps

**Effort 2: Roll Out Across All Middle Office Applications**
Once the MO standard is built, adopt it across every app. This is
an organizational change management effort, not just engineering:
- Team-by-team migration plan
- Adoption tracking dashboard (% of apps on MO standard)
- Onboarding support and documentation for dev teams
- Timeline and prioritization (high-incident apps first?)
- Needs: app inventory, team readiness assessment, rollout schedule

### Terraform / Ansible — not applicable
- Infra provisioning (VMs, DB, MQ) owned by enterprise teams via SNOW — 6–9 month cycle
- Outside Middle Office scope entirely — not a deliverable, not R&D

---

## 9. Chaos Engineering Program [NEEDS DISCUSSION]

- Start with read-only tests: kill one container, measure recovery
- Graduate to quarterly game days with business stakeholders observing
- DR/failover testing with measurable RTO/RPO
- Automated recovery for top-3 known failure modes
- Needs: risk assessment, game day playbook template, stakeholder communication plan

---

## 10. AEv3 Migration & Shared Capacity Pool [DISCUSSED IN DETAIL]

The problem is not "Nomad vs. Kubernetes" — it's that AEv2 (Nomad)
only supports manual scaling. Burst trade processing spikes cause
hours of delayed processing because by the time someone manually
scales, the damage is done. AEv3 (Kubernetes) enables auto-scaling.

### Current state
- AEv2 (Nomad), services run 2–20 instances, right-sized to ~90% utilization
- Hardware paid for up front (internal chargeback), so everyone scales to max
- No room to burst — manual scaling is reactive and too slow
- Burst processing: 10 instances × hours instead of 100 instances × 10 minutes

### Target state
- AEv3 (Kubernetes) with auto-scaling per service (scale to MAX_INSTANCES on load)
- Shared department overhead pool — idle burst capacity funded by all MO apps
- Services auto-scale into the pool when load hits threshold, scale back when done
- Burst processing becomes minutes, not hours — SLAs met

### Business case to build
- Document every incident where burst processing caused delayed trades
- Quantify: how many hours delayed, which clients affected, SLA impact
- Calculate: what auto-scaling would have done (instances × time × cost)
- Propose shared pool funding model — department-wide contribution

### Open questions
- Who approves the budget for the shared overhead pool?
- Per-app overhead vs. shared pool vs. hybrid?
- AEv3 migration effort per service — what does it take to move from V2 to V3?
- Which service to pilot first? (highest burst frequency + impact)
- How does the chargeback model change with a shared pool?

---

## 11. SRE Culture / Enablement [NEEDS DISCUSSION]

- SRE Academy curriculum design (modeled on Standard Chartered)
- SRE Champions program: one trained engineer per dev team
- Production readiness checklist for new services
- Monthly "Reliability Office Hours"
- Quarterly "State of Reliability" report format
- Needs: curriculum outline, champion selection criteria, readiness checklist template

---

## 12. Monday Incident Pattern [DISCUSSED — COVERED BY OTHER ITEMS]

Root cause: weekend changes + human error + lack of QA.
Addressed by: Tenet 1 (SLOs remove severity gaming), Tenet 2
(remediation pipeline), Tenet 3 (automate manual steps), Tenet 4
(CI/CD gates, base image enforcement, post-deploy validation).
May need its own playbook section if the pattern persists after
those controls are in place.
