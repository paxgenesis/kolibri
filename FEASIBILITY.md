# Kolibri Fork Feasibility Assessment
## Trinidad & Tobago Ministry of Education — National Deployment

**Date:** 2026-03-14
**Status:** Revised after critical review (v2)

---

## Concept

A national deployment of Kolibri driven by the Trinidad & Tobago Ministry of Education. Each school is assigned one or more Kolibri servers. Students do lessons, assignments, and assessments from the local server. Results aggregate back to a central MoE database for reporting at student/class/school/district/demographic levels.

**Key assumptions:**
1. Retain offline capability but assume internet connectivity 90%+ of the time
2. Centralized management — school nodes provisioned top-down from central
3. Student user accounts created centrally, assigned at school level
4. Standardized, managed hardware deployed to each school (see [Infrastructure](#infrastructure-baseline))
5. **Data sovereignty** — all student/school data must remain within Trinidad & Tobago jurisdiction. No reliance on external hosted services (including Learning Equality's Kolibri Data Portal)

---

## TL;DR

**Feasibility: High — but a fork may not be needed.** Kolibri has significantly more built-in capability than our initial investigation identified. Stock Kolibri-to-Kolibri sync works directly between any two instances without external services — no dependency on the Kolibri Data Portal (KDP), which is unavailable for this deployment due to data sovereignty requirements. The coach reporting APIs, custom demographics system, and scriptable provisioning cover much of the requirement. The primary question is not "can we fork Kolibri to do this" but "how much of this can stock Kolibri + configuration + a separate reporting layer achieve without forking."

**Recommended approach: Configuration-first, fork-last.** See [Alternative Approach](#alternative-approach-no-fork) below.

### Data Sovereignty Constraint

The Kolibri Data Portal (KDP) operated by Learning Equality is not freely available, is planned as a paid service, and — critically — is hosted outside Trinidad & Tobago. MoE data governance requirements prohibit student PII from leaving TnT jurisdiction. **KDP is therefore excluded from this architecture.**

This is not a blocker. Investigation of the codebase confirms that KDP is **purely an optional registration/discovery service** — it is not part of the sync mechanism. Two stock Kolibri instances can sync directly via the `kolibri manage sync --baseurl <url>` command using P2P certificate exchange. Trust is established by admin credential authentication during initial certificate signing, with no external authority required.

The central server is simply **another stock Kolibri instance hosted on infrastructure within TnT** (government datacenter or local cloud provider).

---

## What Already Exists (More Than We Thought)

| Requirement | Existing Feature | Location | Notes |
|---|---|---|---|
| Offline-first operation | Core design assumption | Throughout | |
| Sync when connected | Morango full-facility sync | `core/auth/constants/morango_sync.py` | Use full-facility, NOT SoUD for this topology |
| School → Central data flow | P2P full-facility sync (direct, no KDP needed) | `core/auth/management/commands/sync.py` | Any two Kolibri instances can sync via `--baseurl <url>` with admin credentials |
| Scheduled sync | SyncSchedule model + UI | `core/device/models.py` | Admin UI for recurring sync schedules exists |
| Per-student activity logs | `ContentSessionLog`, `ContentSummaryLog` | `core/logger/models.py` | |
| Assessments/exams | `Exam`, `ExamAssignment`, attempt tracking | `core/exams/models.py` | |
| Facility > Classroom > Group hierarchy | Auth models | `core/auth/models.py` | |
| PostgreSQL support | Multi-DB routing | `deployment/default/settings/base.py` | |
| REST API surface | DRF viewsets across all core models | `core/*/api.py` | |
| Plugin architecture | Hook-based registry | `plugins/registry.py` | |
| Content assignment per classroom | `LessonAssignment`, `ExamAssignment` | `core/lessons/`, `core/exams/` | |
| **Coach reports** | ClassSummary API (48 functions), LessonReportViewset, QuizDifficultQuestionsViewset | `plugins/coach/class_summary_api.py`, `plugins/coach/api.py` | Per-learner progress, quiz scores, completion status, help-needed flags |
| **CSV data export** | Session logs, summary logs, user demographics | `core/logger/csv_export.py`, `plugins/facility/` | Async task-based, facility admin UI |
| **Notifications** | LearnerProgressNotification | `core/notifications/models.py` | Tracks resource start/complete, quiz answered, help needed |
| **National student ID** | `FacilityUser.id_number` (CharField 64) | `core/auth/models.py` | Built-in field, no model changes needed |
| **Custom demographics** | `extra_demographics` JSONField + JSON Schema validation | `core/auth/constants/demographics.py` | Facility-level schema, per-user values, syncs via Morango |
| **Scriptable provisioning** | `provisiondevice` management command | `core/device/management/commands/provisiondevice.py` | Accepts JSON config, no wizard needed |
| **Multi-facility per device** | FacilityDataset partitioning | `core/auth/models.py` | Central server can host all facilities natively |
| **Facility policy controls** | 10+ boolean settings per facility | `core/auth/models.py` (FacilityDataset) | learner_can_sign_up, show_download_button, etc. |

---

## Revised Gap Analysis

### 1. Centralized Management (Top-Down School Nodes)

**Current state (corrected):** More exists than initially assessed. The `provisiondevice` management command can script device setup. Direct P2P full-facility sync and scheduled sync are production features. Multi-facility hosting is native. No dependency on KDP — school nodes sync directly to the central Kolibri server via `--baseurl`.

**Actual remaining gaps:**
- **Orchestrated deployment** — need Ansible/scripting to provision 100+ school nodes with the management command + facility config. Not a Kolibri problem; it's a DevOps problem.
- **Node monitoring dashboard** — no existing view of "which schools are online and last synced." Could be built as a separate app reading sync session timestamps from the central PostgreSQL.

**Effort:** Low-Medium. Mostly DevOps, not Kolibri development.

---

### 2. Student User Accounts from Central

**Current state (corrected):** `FacilityUser.id_number` already exists for national student IDs. `extra_demographics` with JSON Schema validation handles arbitrary custom fields (district, grade level, etc.). These fields sync via Morango.

**Actual remaining gaps:**
- **Central user creation workflow** — users can be created on the central server and synced down via full-facility sync, but there's no dedicated "bulk import" UI. CSV import or management command scripting would be needed.
- **Transfer deduplication** — if a student moves schools, their `id_number` could be used to match records, but no automated transfer workflow exists.

**Effort:** Low-Medium. Use existing fields, add a bulk import script.

---

### 3. Aggregated Reporting (School/District/Demographic KPIs)

**Current state (corrected):** Extensive per-facility reporting already exists:
- ClassSummary API returns per-learner progress on every resource and quiz
- CSV export covers session logs, summary logs, and user demographics
- LearnerProgressNotification tracks completion events
- All of this data syncs to central via full-facility sync

**Actual remaining gaps:**
- **Cross-facility aggregation views** — the ClassSummary API is scoped to one classroom. Need queries that span all facilities on the central server (e.g., "Grade 8 math completion rate across all schools"). Since all data lands in one PostgreSQL on the central server, this is a query/view layer problem.
- **MoE dashboard** — a reporting UI that presents cross-facility KPIs. This could be a Kolibri plugin OR a separate application (Metabase/Superset/custom Django app) pointed at the central Kolibri database.

**Effort:** Medium. The data is there; build the aggregation queries and a dashboard.

---

### 4. Content/Assessment Push from Central

**Current state (corrected):** Content channels come from Kolibri Studio. Exams are created locally by coaches within a facility. Full-facility sync is bidirectional — data created on the central server (including exams) will sync down to school nodes.

**Actual remaining gaps:**
- **Central exam authoring** — an MoE admin could create exams on the central Kolibri server within each facility. Full-facility sync would push them down. However, this means creating the exam N times (once per facility) or finding a way to template it.
- **Cross-facility exam distribution** — this is the genuinely hard problem. An exam created in Facility A doesn't appear in Facility B. Options:
  - Create the exam on central in every facility (scriptable but clunky)
  - New Morango sync scope for shared content (complex, core change)
  - Separate "exam template" API that school nodes pull and instantiate locally (plugin)
- **Curriculum content pipeline** — who authors TT MoE content? Is it in Studio? This is an operational question, not a code question.

**Effort:** Medium-High. Cross-facility exam distribution remains the hardest technical problem.

---

## Critical Technical Risks (Revised)

### CRITICAL: Morango Sync Concurrency

`MAX_CONCURRENT_SYNCS = 1` in `kolibri/core/public/constants/user_sync_options.py`. The server processes **one sync at a time**. The SoUD queue is per-user, not per-facility.

**Impact at scale:**
- SoUD sync (per-user): 100 schools x 500 students = 50,000 queue entries through a single-concurrency bottleneck. Completely infeasible.
- Full-facility sync (per-school): 100 schools at 1-at-a-time with ~60s intervals = ~100 minutes per complete sync cycle. Tight but potentially workable.

**Mitigations:**
- Use full-facility sync, NOT SoUD, for school-to-central. This reduces 50,000 transactions to 100.
- Raise `MAX_CONCURRENT_SYNCS` to 5-10 on the central server (requires load testing).
- Add jitter to `SYNC_INTERVAL` (default 60s) to prevent thundering herd.
- Consider regional hub topology: 100 schools → 10 regional servers → 1 central (reduces fan-in per server to 10).

**Severity: CRITICAL. Must be load-tested before committing to this architecture.**

### HIGH: Morango Conflict Resolution

Morango flags conflicts with `conflicting_serialized_data` but **does not auto-resolve them**. Winner is determined by instance ID ordering — effectively arbitrary.

**Impact:** If central pushes exams down and schools push logs up, overlapping writes to the same records will generate unresolved conflicts. Must design data flow to be **strictly unidirectional per model type**: central owns exams/users (push down), schools own logs/attempts (push up). Bidirectional writes to the same model = conflict explosion.

**Severity: HIGH. Requires strict data ownership design before implementation.**

### HIGH: Django 3.2 End-of-Life

Django 3.2 LTS ended April 2024 — nearly 2 years of unpatched security vulnerabilities. For a government education platform handling student PII, this is a compliance risk.

Django 3.2 → 4.x upgrade is non-trivial: custom SQLite backend (`kolibri/deployment/default/db/backends/sqlite3/`), multi-database routing, Morango compatibility, extensive ORM usage.

**Severity: HIGH. Either upgrade before deployment or document security exposure for MoE risk acceptance.**

### HIGH: No Sync Performance Data

No load tests or benchmarks exist in the codebase. Unknown how many concurrent full-facility syncs a Kolibri PostgreSQL server can handle, or how long a sync takes for a facility with 500 users and 6 months of logs.

**Severity: HIGH. Must benchmark before deployment commitment.**

### MEDIUM: Cross-Facility Exam Distribution

The only genuinely novel technical problem. No existing mechanism pushes an exam from one facility into another. Solutions range from "script it" (low-tech, high-ops) to "new Morango sync profile" (high-tech, high-risk).

### MEDIUM: Content Pipeline

Who authors TT curriculum content? Is it in Kolibri Studio already? This is an operational dependency, not a code problem, but it's a deployment blocker.

### MEDIUM: Deployment/Operations

100+ school servers need: deployment automation, version updates, monitoring, backups, disaster recovery, network configuration. This is 60% of the actual project work and is not a Kolibri code problem.

---

## Alternative Approach: No Fork

If 90%+ internet connectivity is assumed, consider a zero-fork approach:

```
┌─────────────────────────────────────────────────────────────┐
│  CENTRAL SERVER (TnT datacenter / local cloud)              │
│  Stock Kolibri + kolibri-server (Nginx/uWSGI)               │
│  PostgreSQL — hosts all school facilities (multi-facility)   │
│  Full-facility sync target for all school nodes              │
│  No external service dependencies (no KDP)                   │
└──────────────┬──────────────────────────────────────────────┘
               │ P2P full-facility sync (Morango, scheduled)
               │ HTTPS over internet (all data stays in TnT)
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐  Pre-imaged N100/N305 mini PCs
│School A│ │School B│ │School C│  PostgreSQL (local)
│Kolibri │ │Kolibri │ │Kolibri │  kolibri-server (Nginx/uWSGI)
│  +UPS  │ │  +UPS  │ │  +UPS  │  Provisioned via provisiondevice
└───┬────┘ └────────┘ └────────┘
    │  Local network (Ethernet + WiFi APs)
    ▼
┌──────────────────┐
│ Student Browsers │  Laptops / tablets / desktops
└──────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  REPORTING LAYER (TnT-hosted, separate application)         │
│  Metabase / Superset / custom Django app                    │
│  Reads central PostgreSQL (read-only replica)               │
│  Cross-facility aggregation + MoE dashboards                │
└─────────────────────────────────────────────────────────────┘

All infrastructure hosted within Trinidad & Tobago jurisdiction.
No data leaves TnT. No dependency on external services.
```

**How it works:**
1. Deploy a central Kolibri server (stock) on a VM in a TnT datacenter with PostgreSQL
2. Create one facility per school on the central server
3. Build a base NVMe image: Ubuntu + PostgreSQL + Kolibri + `kolibri-server` + content channels
4. Per-school: run `provisiondevice` with school config + sync target URL (central server), clone image to NVMe, ship hardware
5. School nodes sync directly to central via `kolibri manage sync --baseurl https://central.moe.tt` (P2P full-facility sync, Morango, scheduled)
6. Use `id_number` for national student ID, `extra_demographics` for district/grade/custom fields
7. A separate TnT-hosted reporting app reads the central PostgreSQL and provides MoE dashboards
8. National exams are created on central within each facility and sync down (scriptable)

**Sync trust model (no KDP needed):**
- On first sync, the school node authenticates to the central server using admin credentials
- Morango issues a signed certificate to the school node via certificate signing request (CSR)
- Subsequent syncs use the certificate — no credentials retransmitted
- All certificate authority is the central Kolibri server itself, hosted in TnT

**What this covers without any Kolibri code changes:**
- Offline capability with scheduled sync (full-facility sync)
- PostgreSQL at every tier — school nodes and central (native support, zero fork)
- Centralized data aggregation (all facilities land on one PostgreSQL)
- National student ID (`id_number`)
- Custom demographics per facility (JSON Schema)
- Coach-level reports per classroom (ClassSummary API)
- CSV data export (built-in)
- Automated provisioning (`provisiondevice`)
- Facility policy controls (learner permissions, download buttons, etc.)
- Pre-imaged hardware for plug-and-play school deployment

**What still needs building (but NOT as a Kolibri fork):**
- Reporting dashboard (separate app reading the central DB)
- Deployment automation (Ansible playbooks, not Kolibri code)
- Bulk user import script (management command wrapper)
- Cross-facility exam templating (script to create exam in N facilities)
- Monitoring/alerting (infrastructure, not Kolibri code)

---

## Infrastructure Baseline

### Database: PostgreSQL Everywhere

**Decision: Standardize on PostgreSQL at both school nodes and central server.**

Kolibri defaults to SQLite because it targets zero-administration offline environments with no IT support. That assumption doesn't apply here — we deploy managed, standardized hardware with pre-imaged drives.

**Why not SQLite at school nodes:**
- SQLite is single-writer — only one write transaction at a time. With 30+ concurrent students submitting answers, this becomes a bottleneck.
- Different database engines at school vs. central means two sets of debugging, backup, and operational tooling.
- Kolibri has a custom SQLite backend (`kolibri/deployment/default/db/backends/sqlite3/base.py`) that forces `BEGIN IMMEDIATE` for serializable transactions. PostgreSQL handles this natively with proper isolation levels — cleaner and more robust.

**Why PostgreSQL is free in this context:**
- Installed once in the base drive image. Auto-starts as a systemd service, zero ongoing administration.
- An N100 mini PC with 16GB RAM runs PostgreSQL at negligible overhead for school-scale workloads.
- `pg_dump` / `pg_restore` for backups is battle-tested and scriptable.
- Same engine everywhere = one set of tooling, one set of operational knowledge.

**Configuration (baked into drive image):**
```
DATABASE_ENGINE=postgres
DATABASE_NAME=kolibri
DATABASE_USER=kolibri
DATABASE_PASSWORD=<generated-per-school>
DATABASE_HOST=localhost
DATABASE_PORT=5432
```

**Validation required:** Confirm Morango sync works identically over PostgreSQL as SQLite during Phase 0 testing. Kolibri supports PostgreSQL natively but the sync path should be exercised end-to-end.

---

### Hardware: x86 Mini PCs

**Decision: Intel N100/N150 fanless mini PCs, not Raspberry Pi.**

A Raspberry Pi 5 (8GB) tops out at ~30-40 concurrent Kolibri users with `kolibri-server` (Nginx/uWSGI). For schools with 200+ students and multiple classes hitting the server, that's at the ceiling. The #1 failure mode in Pi deployments is SD card death — accelerated in tropical climates.

**Recommended hardware tiers:**

| School Size | Concurrent Users | Hardware | CPU | RAM | Storage | Cost |
|---|---|---|---|---|---|---|
| Small (< 200 students) | ~30-40 peak | Beelink S12 Pro | Intel N100 4C @3.4GHz | 16GB DDR4 | 256GB NVMe | ~$170 |
| Medium (200-500 students) | ~40-60 peak | MinisForum UN305 | Intel i3-N305 8C @3.8GHz | 16GB DDR5 | 512GB NVMe | ~$300 |
| Large (500+ students) | ~60-100+ peak | 2x N100 load-balanced, or 1x N305 | — | — | — | ~$340-$600 |

**Why x86 mini PC over ARM SBC:**
- **NVMe SSD boot** — no SD card to fail. Image the NVMe, not an SD card.
- **Performance** — N100 at 3.4GHz x86 outperforms Pi 5's A76 cores for Python/Django workloads. N305's 8 cores give real headroom.
- **Fanless aluminium chassis** — acts as heatsink, no fan to clog with dust in tropical climate (Trinidad ambient 28-35C).
- **Total cost parity** — Pi 5 8GB ($80) + case ($15) + PSU ($12) + NVMe HAT ($15) + SSD ($25) + heatsink ($10) = ~$157 assembled. Beelink S12 Pro at $170 comes complete.
- **Standard x86** — Kolibri's Debian packages and `kolibri-server` (Nginx/uWSGI) work out of the box, no ARM compatibility concerns.
- **Supply chain** — multiple manufacturers (Beelink, MinisForum, MINIX). Avoids Raspberry Pi price volatility and stock shortages.

**Tropical climate considerations:**
- Fanless N100 systems handle 35-40C ambient well. Minor thermal throttling possible above 45C (unventilated cabinet). Mitigation: VESA wall-mount, ensure airflow, avoid enclosed cabinets.
- NVMe SSDs have 10x the write endurance of SD cards and no removable connector to fail from vibration/humidity.
- Power consumption: 6-12W typical = UPS-friendly (a 600VA UPS powers one for hours during outages).

**Per-school kit:**

| Component | Example | Cost |
|---|---|---|
| Server | Beelink S12 Pro (16GB/256GB NVMe) | ~$170 |
| Gigabit switch | TP-Link 8-port | ~$20 |
| UPS | APC 600VA | ~$50 |
| **Total** | (excluding WiFi APs and client devices) | **~$240** |

WiFi access points and broader networking handled separately.

---

### Server Software Stack

Each school node runs the same pre-imaged software stack:

| Layer | Component | Notes |
|---|---|---|
| OS | Ubuntu 22.04 LTS (or 24.04) | Long-term support, automatic security updates |
| Database | PostgreSQL 15+ | Local instance, auto-start via systemd |
| Application | Kolibri (stock, latest stable) | Installed via `kolibri` Debian package |
| HTTP frontend | `kolibri-server` (Nginx + uWSGI) | 2-3x more client capacity than stock CherryPy server |
| Cache | Redis (optional, for larger schools) | Shared cache across uWSGI workers |

**Kolibri performance tuning (in image):**
- `CHERRYPY_THREAD_POOL`: Not applicable when using `kolibri-server` (Nginx/uWSGI replaces CherryPy)
- `kolibri-server` uses uWSGI workers — configure 4-8 workers based on CPU cores
- PostgreSQL `shared_buffers`: 2GB (of 16GB RAM) is ample for school scale
- Content pre-loaded on the NVMe before deployment

---

### Deployment Model: Pre-Imaged Drives

**Process:**
1. Build a base NVMe image with: Ubuntu + PostgreSQL + Kolibri + `kolibri-server` + content channels
2. Per-school customization: run `provisiondevice` with school-specific config (facility name, admin credentials, sync target URL, demographic schema)
3. Clone the customized image to the school's NVMe drive
4. Ship hardware to school, plug in, power on — zero on-site configuration

**Field replacement:** If a unit fails, ship a replacement with a fresh pre-imaged drive. The school's data lives on the central server (synced). The replacement node syncs down its facility data on first connection and is operational within minutes.

**Version updates:** Managed remotely via `apt upgrade` (Kolibri Debian package) + Ansible. Since internet is assumed 90%+ of the time, unattended-upgrades can handle routine security patches.

---

## Approach Comparison

| Factor | Fork Approach | No-Fork Approach |
|---|---|---|
| Core code changes | 4+ areas (user model, sync scopes, wizard, hierarchy) | Zero |
| Upstream mergeability | Degrades over time | Full compatibility |
| Time to first deployment | 6-12 months (estimate) | 2-4 months (estimate) |
| Reporting | Custom Kolibri plugin | Separate app (Metabase, Superset, or custom) |
| Cross-facility exams | New Morango sync scope | Script to create exam per facility |
| Risk profile | High (untested sync topology, core changes, Django EOL) | Lower (stock software, proven sync, isolated reporting) |
| Maintenance burden | Must track upstream + maintain fork patches | Stock Kolibri + separate reporting app |
| Upgrade path | Every upstream release requires merge conflict resolution | Standard Kolibri upgrades + separate app updates |

---

## Recommended Next Steps (Revised)

### Phase 0: Validate (before committing to any approach)
1. **Stand up two Kolibri instances on PostgreSQL** and test P2P full-facility sync (no KDP). Use `kolibri manage sync --baseurl <url>`. Measure: sync duration for a facility with 500 users and 3 months of logs. Test with `MAX_CONCURRENT_SYNCS` at 1, 5, and 10. Confirm Morango sync works identically over PostgreSQL as SQLite.
2. **Test certificate trust flow** — confirm that initial CSR (admin credentials) + subsequent cert-based sync works cleanly between two stock Kolibri instances. No external certificate authority needed.
3. **Test `provisiondevice` with PostgreSQL** — script a fully automated school node setup on the target hardware (N100 mini PC). Confirm it works without human interaction. Build a reproducible NVMe image.
4. **Test `id_number` + `extra_demographics`** — create a JSON Schema for TT demographics, provision users with national IDs, confirm it syncs over PostgreSQL via P2P sync.

### Phase 1: Deploy stock Kolibri (if Phase 0 validates)
5. **Deploy 3-5 pilot schools** using the no-fork approach with stock Kolibri + Ansible.
6. **Build reporting layer** — connect Metabase or similar to the central PostgreSQL. Build the 5-10 KPI dashboards MoE needs.
7. **Build bulk user import** — management command or script to create users from MoE student roster CSV.

### Phase 2: Evaluate fork necessity (only if Phase 1 reveals blockers)
8. **Cross-facility exam distribution** — if scripting exams per-facility is operationally unacceptable, then scope a Morango sync profile extension.
9. **Sync concurrency** — if full-facility sync at 100 schools saturates the central server, consider regional hub topology or a custom ETL pipeline.
10. **Django upgrade** — if MoE compliance requires patched Django, budget this as a standalone project.

---

## Codebase Reference

Key files for implementation:

| Area | File |
|---|---|
| Sync concurrency limit | `kolibri/core/public/constants/user_sync_options.py` |
| Sync queue & scheduling | `kolibri/core/device/models.py` (SyncQueue, SyncSchedule) |
| SoUD sync orchestration | `kolibri/core/device/soud.py` |
| Sync scopes & states | `kolibri/core/auth/constants/morango_sync.py` |
| Sync operations | `kolibri/core/auth/sync_operations.py` |
| User/facility models | `kolibri/core/auth/models.py` |
| Custom demographics | `kolibri/core/auth/constants/demographics.py` |
| Device provisioning command | `kolibri/core/device/management/commands/provisiondevice.py` |
| Session & summary logs | `kolibri/core/logger/models.py` |
| CSV export | `kolibri/core/logger/csv_export.py` |
| Coach class summary API | `kolibri/plugins/coach/class_summary_api.py` |
| Coach report viewsets | `kolibri/plugins/coach/api.py` |
| Notifications | `kolibri/core/notifications/models.py` |
| Exam models | `kolibri/core/exams/models.py` |
| Sync management command | `kolibri/core/auth/management/commands/sync.py` (P2P sync via --baseurl) |
| Sync utilities (cert exchange) | `kolibri/core/auth/management/utils.py` (get_client_and_server_certs) |
| Static network locations | `kolibri/core/discovery/models.py` (StaticNetworkLocation for sync target config) |
| Portal integration (NOT USED) | `kolibri/core/utils/portal.py` (KDP — excluded due to data sovereignty) |
| Public/sync APIs | `kolibri/core/public/api.py` |
| Network client | `kolibri/core/discovery/utils/network/client.py` |
| Facility settings | `kolibri/core/auth/models.py` (FacilityDataset) |
| Device settings | `kolibri/core/device/models.py` (DeviceSettings) |
| Conflict handling | Morango `Store.conflicting_serialized_data` (in morango package) |
| Sync interval config | `kolibri/utils/options.py` (SYNC_INTERVAL) |
| Plugin registry | `kolibri/plugins/registry.py` |
| Database config | `kolibri/deployment/default/settings/base.py` |
| PostgreSQL settings | `kolibri/utils/options.py` (DATABASE_ENGINE, DATABASE_NAME, etc.) |
| Custom SQLite backend | `kolibri/deployment/default/db/backends/sqlite3/base.py` (not used with PostgreSQL) |
| CherryPy thread pool config | `kolibri/utils/options.py` (CHERRYPY_THREAD_POOL — replaced by uWSGI when using kolibri-server) |
| kolibri-server (Nginx/uWSGI) | External package: `github.com/learningequality/kolibri-server` |
| Integration tests (sync) | `integration_testing/019-features/superadmin/device/device-facilities/` |
