# EPA Community Archive — Digital Preservation Platform

> **Note:** This is a production application built for a community and environmental advocacy
> client. The source code is private. This document outlines the architecture,
> engineering decisions, and key challenges solved during development and
> infrastructure stabilization.

---

## 📋 Overview & Problem

Environmental justice organizations generate decades of irreplaceable community
records — oral histories, EPA correspondence, photographs, legal documents. Without
a proper preservation system, these materials exist on aging hard drives,
deteriorating physical media, and fragmented local file systems, at serious risk
of permanent loss.

The client needed a production-grade **digital repository** that could:

- Ingest and preserve thousands of archival items with rich metadata
- Generate searchable, accessible derivatives (thumbnails, previews) automatically
- Serve a public catalog to researchers and community members
- Run reliably at low operational cost on a lean infrastructure budget

---

## 🛠️ Tech Stack

| Layer | Technology |
| --- | --- |
| Framework | Ruby on Rails (Hyrax/Samvera engine) |
| Object Storage | Fedora Commons 4.7.5 |
| Search | Apache Solr 8.11.1 |
| Database | PostgreSQL 16 (AWS RDS) |
| Background Jobs | Sidekiq + Redis |
| Cache | Memcached |
| Web Server | Puma (cluster mode, 3 workers) |
| Containerization | Docker + Docker Compose |
| Cloud Infrastructure | AWS EC2, RDS, EFS, CloudFront, WAF |
| CI/CD | GitLab CI (private macOS runner → AWS EC2) |
| Media Viewer | Universal Viewer |
| Security | AWS WAF (rate limiting, bot mitigation) |

---

## 🏗️ Architecture & Core Features

[CloudFront + AWS WAF]
        │
        ▼
[EC2: Docker — Puma/Rails App]
        │               │
        ▼               ▼
[AWS RDS           [Sidekiq Workers]
 PostgreSQL]              │
                          ▼
                  [Fedora Commons]  ←→  [Apache Solr]
                          │
                          ▼
                    [AWS EFS — Derivatives]

**Key subsystems:**

- **Ingestion pipeline:** Files uploaded to the Rails app are stored as binary
  objects in Fedora Commons using a pair-tree path structure. Metadata is indexed
  in Solr for full-text search.

- **Derivative generation:** Sidekiq workers run `CreateDerivativesJob`
  asynchronously for every ingested FileSet, generating thumbnail JPEGs written
  to AWS EFS. 2,150+ derivatives generated across the full archive.

- **Public catalog:** Blacklight-powered search UI with faceted filtering by
  subject, theme, resource type, and date. Supports English and Spanish.

- **Background job queue:** Redis-backed Sidekiq processes derivative generation,
  file characterization (FITS), and notification jobs without blocking web requests.

---

## 🧠 Architectural Decisions

### Why Hyrax/Samvera instead of a custom Rails app?

The Samvera community has solved the hardest problems in digital preservation —
PCDM data models, Fedora integration, derivative workflows, IIIF compliance —
over a decade. Building these from scratch would have taken years and produced an
inferior result. The trade-off is framework complexity and a steep learning curve,
but for a preservation-focused use case, the ecosystem value far outweighs the cost.

### Why Fedora Commons for object storage instead of S3?

Fedora provides a linked-data-native object store with built-in provenance tracking,
versioning, and PCDM compliance — all requirements for archival integrity. S3 stores
bytes; Fedora stores objects with semantic relationships. For a permanent archive,
that distinction matters. S3 is used for backups, not primary storage.

### Why PostgreSQL on RDS instead of Docker-managed Postgres?

Early in the project, the database ran inside a Docker container. A production
incident where a container restart caused data-at-risk made it clear that
mission-critical relational data should not live in a container volume. Migrating
to AWS RDS gave us automated backups, Multi-AZ failover capability, and eliminated
an entire class of operational risk.

### Why Puma cluster mode with `puma_worker_killer`?

A single Puma process on a 7.3GB EC2 instance became a bottleneck under concurrent
requests. Switching to cluster mode with 3 workers added parallelism. However,
Ruby's memory model causes Puma workers to accumulate heap over time
("memory bloat"). `puma_worker_killer` monitors each worker every 30 seconds and
restarts any that exceed 1GB — replacing unpredictable manual restarts with
automated, rolling recovery invisible to users.

---

## ⚙️ Challenges & Engineering Wins

### 1. Derivative Generation at Scale

**Problem:** 2,175 archived FileSets had no thumbnail derivatives — they were
ingested before the derivative pipeline was operational, leaving the public catalog
showing broken images.

**Solution:** Provisioned a temporary fleet of 6 parallel Docker worker containers
on a spot EC2 instance, each running `CreateDerivativesJob` against a Fedora
snapshot. Identified a gap of 74 FileSets caused by non-deterministic Solr sort
order between batch workers, resolved with a targeted script using hardcoded FileSet
IDs. Result: **2,150 derivatives generated in a single batch run** at minimal cost.
Worker EC2 was a spot instance auto-terminated on completion.

### 2. Production Memory Bloat → Automated Self-Healing

**Problem:** Puma workers grew from ~400MB at startup to 3GB+ after 4–6 hours,
exhausting available RAM and forcing the server into heavy swap usage. Required
manual intervention every few hours to force-restart containers.

**Solution:** Implemented `puma_worker_killer` gem configured to restart any worker
exceeding 1GB, checking every 30 seconds. Workers now self-manage: after 17+ hours
of uptime, workers hold at 440–550MB. **Zero manual interventions required since
deployment.**

### 3. Bot Attack → WAF Rate Limiting

**Problem:** Automated crawlers saturated all 3 Puma workers simultaneously by
hammering the `/catalog` search endpoint, causing legitimate users to queue behind
bot traffic. The WAF existed but was misconfigured (default action: Block,
accidentally blocking all traffic).

**Solution:** Fixed WAF default action to Allow, then added a rate-limit rule
capping any single IP to 100 requests per 5 minutes on `/catalog*`, returning 429.
Attached WAF to CloudFront distribution (was previously unenforced). **Bot traffic
dropped to negligible levels within minutes of deployment.**

### 4. Unbounded Docker Log Growth → Disk Exhaustion

**Problem:** Production disk reached 97% capacity (2.9GB free) within 2 days of
deployment due to Docker's default behavior of writing container logs to a single
unbounded JSON file. The `hyrax-app` container alone generated 29GB of logs in
under 48 hours.

**Solution:** Added explicit `logging` configuration to `docker-compose.yml`
capping each container at 100MB per file with 3 rotating files (300MB total max).
**Disk stabilized at ~65–75% with no further manual intervention.**

---

## 🏆 Key Achievements & Impact

- **2,150+ archival items** with accessible thumbnails served to the public catalog
- **Zero memory-related outages** since `puma_worker_killer` deployment
- **99%+ uptime** maintained through Rack::Timeout, WAF protection, and
  automated worker management
- **Disk usage stabilized** from crisis-level 97% to sustainable 65–75% through
  log rotation and image pruning automation
- **Automated CI/CD pipeline** from GitLab commit to AWS EC2 production deploy
  with zero manual SSH steps
- **Multi-language catalog** serving English and Spanish-speaking communities

---

## 👤 My Specific Contributions

This project was built by a cross-functional team of four engineers, a UX designer, and a Project Manager. The UX designer led primary research, developed personas and user journeys, and informed the information architecture. The Project Manager used an agile methodology to coordinate delivery across phases.

As the lead engineer, I authored the technical requirements and technical design documents, and collaborated with the Project Manager to write user stories and distribute project phases across the engineering team and UX designer.

My engineering work owned:

- Full AWS infrastructure management (EC2, RDS, EFS, WAF, CloudFront)
- CI/CD pipeline design and maintenance (GitLab CI, Docker buildx, multi-stage builds)
- Production incident response and root-cause analysis
- Derivative generation pipeline design and batch execution
- Memory management strategy (Puma cluster mode + worker killer)
- Security hardening (WAF rules, rate limiting, bot mitigation)
- Operational automation (log rotation, Docker image lifecycle, disk management)

---

## 🔬 Deep Dive: Derivative Generation at Scale {#derivative-generation-deep-dive}

The challenge was regenerating 2,175 missing thumbnail derivatives without
disrupting the live production system or paying for excessive compute time.

**Approach:**

1. Took a point-in-time snapshot of the production Fedora/Solr stack onto a
   separate EC2 instance (`hyraxstack--data-copy`)
2. Provisioned a temporary c5.2xlarge spot instance with 6 parallel Docker
   workers, each running the same `ghcr.io/samvera/dassie` image as production
3. Split the 2,175 FileSet IDs across 6 workers using a modulo-based batch
   script (`bundle exec rails runner run_batch.rb N 6`)
4. Workers wrote thumbnail JPEGs to a shared AWS EFS mount at the pair-tree path:
   `/derivatives/{aa}/{bb}/{cc}/{dd}/{aabbccdd...}-thumbnail.jpeg`
   where the path is derived from the FileSet ID using:

   ```ruby
   fs_id.chars.each_slice(2).map(&:join).join("/")
   ```

5. Post-run analysis revealed 74 FileSets were missed due to Solr returning
   results in non-deterministic order between batch boundaries. Fixed with a
   targeted script using hardcoded IDs from the production catalog page.

**Key insight:** ActiveRecord::ConnectionNotEstablished errors appeared in worker
logs for every job. Initial concern was that derivatives weren't being written.
Investigation confirmed the error fires after the JPEG is written to EFS, during
a metadata write-back to PostgreSQL (unreachable from the worker subnet). All 2,150
derivatives landed correctly on EFS despite the error logs.
