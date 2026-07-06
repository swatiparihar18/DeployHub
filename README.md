# DeployHub — Product & Architecture Design Document

**A self-hosted-friendly PaaS competing with Railway / Render for small-to-medium deployments**

Version 1.0 · Pre-Development Design Doc

**Infrastructure decision (confirmed):** Hybrid rollout. Phase 1–2 run on **Docker Compose + Nginx + PM2 on a single EC2 host** for speed to market and simplicity. The architecture is designed **K8s-ready** from day one (stateless app containers, externalized state, declarative service definitions) so that Phase 4+ can migrate the orchestration layer to Kubernetes (EKS or self-managed) without redesigning the data model, API, or queue architecture. Sections below call out the seams where this matters.

---

## Table of Contents

1. Product Vision
2. Functional Requirements
3. User Flow
4. Complete Frontend Design
5. Backend Design
6. Database Design
7. REST API Design
8. Background Jobs
9. Docker Architecture (+ K8s Migration Path)
10. Security
11. Monitoring
12. Project Folder Structure
13. Development Roadmap

---

## 1. Product Vision

### 1.1 Why This Product Exists

Railway and Render solved a real problem — "git push and get a URL" — but they did it as closed, hosted-only SaaS. That leaves a gap for teams who want the same developer experience with:

- **Cost predictability at scale.** Usage-based pricing on Railway/Render becomes expensive once traffic grows; teams outgrow the free/hobby tier fast and have no easy path to run the same platform on their own infrastructure.
- **Data residency / compliance control.** Regulated teams (fintech, healthcare, EU-based companies) often cannot use a third-party PaaS for compliance reasons and are forced back to raw AWS/GCP consoles — a huge DX regression.
- **Vendor lock-in avoidance.** Once a team's deploy config, secrets, and domain routing live entirely inside a closed SaaS, migrating away is costly. An open, self-hostable core mitigates this.

DeployHub is positioned as **"Railway's developer experience, deployable on your own cloud account."** The managed/hosted version funds development; the self-hosted version is the wedge that gets adoption inside cost-sensitive and compliance-sensitive teams.

### 1.2 Problems It Solves

| Problem | Current Pain | DeployHub Solution |
|---|---|---|
| Manual server setup | Devs hand-roll Nginx, PM2, SSL, Docker on EC2 | One-click GitHub-connected deploy pipeline |
| Zero-downtime deploys | Manual reverse-proxy swaps, error-prone | Automated build → health-check → traffic switch |
| No rollback safety net | Bad deploy = manual SSH firefighting | One-click rollback to any previous build |
| Fragmented observability | Logs on one box, no metrics dashboard | Centralized logs (Loki) + metrics (Prometheus/Grafana) |
| SSL renewal toil | Cron jobs, forgotten renewals, expired certs | Automatic Let's Encrypt issuance + renewal |
| Environment/secret sprawl | `.env` files copied over SSH | Encrypted, versioned env var management per environment |
| Vendor lock-in | SaaS-only PaaS | Self-hostable core, open deployment format |

### 1.3 Target Users

| Segment | Profile | Why DeployHub |
|---|---|---|
| **Indie hackers / solo devs** | Ship side projects fast, cost-sensitive | Free/cheap tier, GitHub-native flow |
| **Small startup engineering teams (2–15 devs)** | No dedicated DevOps hire | Managed hosted plan removes ops burden |
| **Agencies** | Deploy many small client apps | Multi-tenant orgs, per-project isolation |
| **Compliance-conscious mid-market teams** | Need infra inside their own AWS account | Self-hosted / BYO-cloud deployment mode |

### 1.4 Competitive Advantages

1. **BYO-infrastructure mode** — deploy the control plane into the customer's own AWS account; Railway/Render don't offer this.
2. **Transparent, inspectable pipeline** — Docker Compose/K8s manifests are visible and exportable, not a black box.
3. **Built-in rollback-first design** — every deployment is immutable and addressable; rollback is a first-class object, not an afterthought.
4. **Progressive infra complexity** — starts on a single EC2 box (cheap, simple) and scales to Kubernetes without a platform rewrite.
5. **Observability included by default** (Prometheus/Grafana/Loki), not a paid add-on.

### 1.5 Future Roadmap (High-Level)

- **Now → Phase 3:** Node.js support, single-region, Docker Compose orchestration.
- **Phase 4:** Multi-language buildpacks (Python, Go, Java, Rust), Dockerfile-native apps, static sites.
- **Phase 5:** Kubernetes-based orchestration option, multi-region, horizontal autoscaling.
- **Phase 6:** Billing/metering, marketplace add-ons (databases, Redis, object storage), preview environments per PR.
- **Phase 7:** Self-hosted "DeployHub Enterprise" distribution (Helm chart / Terraform module) for customers who want the control plane in their own VPC.

---

## 2. Functional Requirements

Legend: **P0** = required for MVP (Phase 1–2), **P1** = near-term (Phase 3–4), **P2** = later (Phase 5+).

### 2.1 Authentication & Accounts

| Feature | Priority |
|---|---|
| GitHub OAuth login (sole login method for MVP) | P0 |
| JWT access token + refresh token rotation | P0 |
| Session management / "sign out of all devices" | P0 |
| Personal Access Tokens (API tokens) for CLI/CI use | P0 |
| Two-factor auth (via GitHub, inherited) | P0 |
| Email/password login | P2 |
| SSO (SAML/OIDC) for Enterprise | P2 |

### 2.2 Organizations & Teams

| Feature | Priority |
|---|---|
| Personal account (implicit org of one) | P0 |
| Create/rename/delete organizations | P0 |
| Invite members by email/GitHub username | P0 |
| Roles: Owner, Admin, Member, Viewer | P0 |
| Per-project role overrides | P1 |
| Transfer project between orgs | P1 |
| Org-level audit log | P1 |
| SCIM provisioning | P2 |

### 2.3 GitHub Integration

| Feature | Priority |
|---|---|
| OAuth App connection | P0 |
| List accessible repos (personal + org, respecting GitHub permissions) | P0 |
| Repo search/filter in import UI | P0 |
| Auto-create webhook on import (push, PR events) | P0 |
| Branch selection (deploy branch) | P0 |
| Auto-deploy on push to selected branch | P0 |
| PR preview deployments | P1 |
| Monorepo support (subdirectory as root) | P1 |
| GitHub Checks API status reporting (deploy status on commit) | P1 |
| GitHub App (fine-grained permissions) migration from OAuth App | P2 |

### 2.4 Projects

| Feature | Priority |
|---|---|
| Create project from repo import | P0 |
| Auto-detect framework/runtime (Node.js: package.json, start script) | P0 |
| Manual build/start command override | P0 |
| Multiple environments per project (production, staging, preview) | P1 |
| Project settings (name, root directory, build config) | P0 |
| Archive/delete project | P0 |
| Project-level transfer of ownership | P1 |
| Static site / Dockerfile-based project type | P1 |

### 2.5 Environment Variables & Secrets

| Feature | Priority |
|---|---|
| Add/edit/delete env vars per environment | P0 |
| Encrypted at rest (AES-256-GCM, KMS-managed key) | P0 |
| Bulk import via `.env` paste | P0 |
| Reveal/hide values in UI (masked by default) | P0 |
| Secret vs. plain variable distinction (secrets never shown again after save) | P1 |
| Variable diffing across deployments | P1 |
| Shared/team-level variable groups | P2 |

### 2.6 Deployments

| Feature | Priority |
|---|---|
| Trigger deploy manually | P0 |
| Auto-deploy on push (per branch) | P0 |
| Build pipeline: clone → install → build → containerize | P0 |
| Real-time build logs via WebSocket | P0 |
| Deployment states: queued, building, deploying, live, failed, cancelled | P0 |
| Cancel in-progress deployment | P0 |
| Deployment history list (immutable records) | P0 |
| Redeploy (re-run build for a commit) | P0 |
| Rollback (promote a previous successful build without rebuilding) | P0 |
| Zero-downtime deploy (build new container, health-check, then switch traffic) | P0 |
| Deployment-level environment variable snapshot | P1 |
| Concurrent build limits per plan | P1 |
| Build cache (layer caching) | P1 |

### 2.7 Logs

| Feature | Priority |
|---|---|
| Live build logs (streamed) | P0 |
| Live runtime/application logs (streamed) | P0 |
| Historical log search (last N days, retention by plan) | P1 |
| Log filtering (by deployment, severity, time range) | P1 |
| Log export/download | P1 |
| Centralized aggregation via Loki | P1 |

### 2.8 Domains & SSL

| Feature | Priority |
|---|---|
| Auto-generated subdomain per project (`project.deployhub.app`) | P0 |
| Add custom domain | P0 |
| Domain verification (DNS TXT/CNAME check) | P0 |
| Automatic SSL cert issuance (Let's Encrypt / ACME) | P0 |
| Auto-renewal + expiry monitoring | P0 |
| Multiple domains per project | P1 |
| Wildcard domain / apex domain support | P1 |
| Redirect rules (www → apex, http → https forced) | P0 |

### 2.9 Health Checks & Scaling

| Feature | Priority |
|---|---|
| HTTP health check endpoint config | P0 |
| Health-gated traffic cutover during deploy | P0 |
| Auto-restart on crash (PM2 / container restart policy) | P0 |
| Manual scale (replica count) | P1 |
| Resource limits (CPU/memory caps per plan) | P0 |
| Autoscaling based on CPU/RPS | P2 (K8s phase) |

### 2.10 Monitoring & Metrics

| Feature | Priority |
|---|---|
| CPU/memory/disk usage per deployment | P0 |
| Request count / latency metrics | P1 |
| Uptime tracking | P0 |
| Historical charts (Grafana embeds) | P1 |
| Alerting (email/webhook on downtime or resource threshold) | P1 |
| Custom alert rules | P2 |

### 2.11 Notifications

| Feature | Priority |
|---|---|
| In-app notification center | P1 |
| Email notifications (deploy success/fail, domain expiry, SSL renewal) | P0 |
| Slack/Discord webhook integration | P1 |
| Notification preferences per user | P1 |

### 2.12 API Tokens & API Access

| Feature | Priority |
|---|---|
| Create/revoke personal API tokens | P0 |
| Scoped token permissions (read-only, deploy-only, full) | P1 |
| Token last-used tracking | P1 |
| Public REST API parity with UI actions | P0 |

### 2.13 Billing (Future)

| Feature | Priority |
|---|---|
| Plan tiers (Free, Pro, Team, Enterprise) | P2 |
| Usage metering (build minutes, bandwidth, compute hours) | P2 |
| Stripe integration | P2 |
| Invoices & payment history | P2 |
| Usage alerts / hard caps | P2 |

### 2.14 Audit Logs & Admin

| Feature | Priority |
|---|---|
| Org-level audit trail (who did what, when) | P1 |
| Platform admin panel (internal ops) | P1 |
| System-wide health dashboard (internal) | P1 |
| Feature flags | P2 |
| Impersonation for support (with audit trail) | P2 |

### 2.15 System Settings

| Feature | Priority |
|---|---|
| User profile management (name, avatar, GitHub identity) | P0 |
| Org settings (name, logo, default region) | P0 |
| Notification settings | P1 |
| Danger zone (delete org/project, transfer ownership) | P0 |

---

## 3. User Flow

### 3.1 Primary Journey: New User → Live App

```
Landing Page
    │
    ▼
"Sign in with GitHub" ──▶ GitHub OAuth consent ──▶ Callback ──▶ JWT issued
    │
    ▼
Dashboard (empty state: "Import your first project")
    │
    ▼
Import Repository
    ├─ Select GitHub org/account
    ├─ Search/filter repo list
    ├─ Select repo + branch
    └─ Click "Import"
    │
    ▼
Configure Build
    ├─ Auto-detected: Node.js, build cmd, start cmd, port
    ├─ Override root directory (monorepo)
    ├─ Add environment variables (paste .env or add manually)
    └─ Select environment (Production)
    │
    ▼
Deploy (click "Deploy" or auto-triggered)
    ├─ Job enters Build Queue (BullMQ)
    ├─ Status: Queued → Building
    └─ Real-time log stream opens (Socket.IO)
    │
    ▼
Logs (build phase)
    ├─ Clone repo
    ├─ Install dependencies
    ├─ Run build command
    ├─ Build Docker image
    └─ Push to internal registry
    │
    ▼
Deploying phase
    ├─ Start new container
    ├─ Wait for health check to pass
    ├─ Switch Nginx upstream to new container
    └─ Stop old container (grace period for in-flight requests)
    │
    ▼
Running Application
    ├─ Status: Live
    ├─ URL shown: https://project.deployhub.app
    ├─ Runtime logs streaming
    └─ Metrics tile (CPU/mem/uptime)
    │
    ▼
   ┌──────────────┬──────────────────┬───────────────┐
   ▼              ▼                  ▼               ▼
Redeploy      Rollback          Add Domain      Delete Project
(new commit   (pick prior       (custom domain   (confirm dialog,
 or same       successful        + auto SSL)      cascades cleanup)
 commit)       build, instant
               traffic switch,
               no rebuild)
```

### 3.2 Flow Detail: Redeploy vs. Rollback (important distinction)

| | Redeploy | Rollback |
|---|---|---|
| Trigger | New commit pushed, or manual "Redeploy" on existing commit | User selects a prior **successful** deployment |
| Rebuild? | Yes — full pipeline runs again | No — reuses the already-built, immutable image |
| Speed | Minutes (build time) | Seconds (image already exists, just re-point traffic) |
| Use case | Ship new code, or retry a failed build | Bad deploy in production, need instant revert |
| Downtime | Zero (health-checked cutover) | Zero (health-checked cutover) |

### 3.3 Flow Detail: Team Invite

```
Org Settings → Members → "Invite Member"
    → Enter email or GitHub username → Select role (Admin/Member/Viewer)
    → Invite email sent → Invitee clicks link → GitHub OAuth (if not logged in)
    → Invitee lands in org with assigned role
```

### 3.4 Flow Detail: Custom Domain + SSL

```
Project → Domains → "Add Domain"
    → Enter domain (e.g. app.customer.com)
    → DeployHub shows required DNS record (CNAME or A)
    → User adds record at their DNS provider
    → "Verify" → DeployHub polls DNS
    → On success: SSL Queue job triggers ACME HTTP-01 challenge
    → Certificate issued → Nginx config reloaded with cert
    → Domain status: Active (green), auto-renewal scheduled at 60 days
```

### 3.5 Error / Edge-Case Flows

- **Build failure** → status = Failed, logs retained, user notified (email + in-app), "Retry" button available, previous deployment stays live (no partial traffic switch).
- **Health check never passes** → deployment marked Failed after timeout (default 5 min, configurable), old container remains serving traffic, alert fired.
- **Domain verification failure** → clear inline error with the exact expected DNS record vs. what was found, retry button, no destructive action taken.
- **Concurrent deploys on same project** → second deploy queued, not run in parallel, to avoid container/port collisions on the single-host phase.

---

## 4. Complete Frontend Design

**Stack:** Next.js (App Router) + TypeScript + TailwindCSS + shadcn/ui + React Query (server state) + Zustand (client/UI state) + Framer Motion (transitions).

**Global conventions applied to every page below:**
- **Loading state:** shadcn `Skeleton` components matching the final layout's shape (never a bare spinner for content areas; spinners only for button-level actions).
- **Empty state:** icon + one-line explanation + primary CTA.
- **Error state:** inline `Alert` (destructive variant) with retry action; toast for transient errors, full-page error boundary for fatal ones.
- **Responsive behavior:** Tailwind breakpoints `sm/md/lg/xl`; sidebar collapses to a bottom nav / hamburger drawer below `md`; tables convert to stacked cards below `md`.
- **Animations:** Framer Motion page transitions (fade+slide, 150ms), list item enter/exit, modal scale-in.
- **Permissions:** every mutating action checks role client-side (hide/disable) **and** server-side (never trust client checks alone).

### 4.1 Landing Page

| Aspect | Detail |
|---|---|
| Purpose | Convert visitors: explain value prop, drive to GitHub sign-in |
| Components | Nav bar, hero with animated deploy-flow illustration, feature grid (3–4 cards), pricing teaser, testimonials, footer |
| Buttons | "Sign in with GitHub" (primary, nav + hero), "View Docs", "Star on GitHub" |
| Responsive | Hero stacks vertically on mobile; feature grid 3-col → 1-col |
| Loading/Empty/Error | Static marketing page — no data-dependent states |
| Animations | Scroll-reveal on feature cards, hero illustration subtle float loop |
| Permissions | Public, unauthenticated |

### 4.2 Dashboard (Org Home)

| Aspect | Detail |
|---|---|
| Purpose | At-a-glance view of all projects + recent activity across the org |
| Components | Org switcher (top-left dropdown), project grid/list toggle, "New Project" button, recent deployments feed, usage summary card |
| Cards | Per-project card: name, framework icon, latest deploy status badge, URL, last deployed time |
| Modals | "New Project" opens repo import modal |
| Filters | Search by project name, filter by status (live/failed/building) |
| Empty state | "No projects yet" illustration + "Import your first repository" CTA |
| Loading state | Skeleton grid of project cards |
| Error state | Alert banner if project list fetch fails, with retry |
| Permissions | Viewer: read-only; Member+: can create projects |

### 4.3 Import Repository (Modal / Page)

| Aspect | Detail |
|---|---|
| Purpose | Connect a GitHub repo and create a project |
| Components | GitHub account/org selector, searchable repo list (virtualized for large orgs), branch selector |
| Forms | Search input, repo list (radio-select rows) |
| Empty state | "No repositories found" + "Configure GitHub App access" link if permission-limited |
| Error state | GitHub API rate-limit or auth error shown with re-auth CTA |
| Loading | Skeleton rows while fetching repo list |
| Permissions | Member+ |

### 4.4 Configure Build (Step within import flow)

| Aspect | Detail |
|---|---|
| Purpose | Set build/start commands, root dir, env vars before first deploy |
| Components | Auto-detected framework badge, editable command fields, root directory input, env var table with add-row, `.env` paste-to-import textarea |
| Forms | Build command, install command, start command, port, root directory |
| Validation | Port must be numeric 1–65535; commands non-empty; env var keys must match `[A-Z_][A-Z0-9_]*` |
| Buttons | "Deploy" (primary), "Save as draft" |
| Permissions | Member+ |

### 4.5 Project Details (Overview Tab)

| Aspect | Detail |
|---|---|
| Purpose | Central hub for one project: status, URL, quick actions |
| Components | Header with project name/URL/status badge, tab bar (Overview / Deployments / Logs / Env Vars / Domains / Settings), latest deployment card, metrics mini-charts (CPU/mem sparkline), quick actions (Redeploy, Restart, Rollback) |
| Buttons | Redeploy, Restart, Rollback (dropdown of prior builds), Delete Project (danger zone) |
| Charts | CPU/memory sparkline (last 1h), request count sparkline |
| Loading | Skeleton header + skeleton chart placeholders |
| Empty state | "No deployments yet" if project created but never deployed |
| Permissions | Viewer: read-only; Member+: Redeploy/Restart; Admin+: Rollback/Delete |

### 4.6 Deployment History

| Aspect | Detail |
|---|---|
| Purpose | List every deployment, immutable audit trail |
| Table columns | Commit (SHA + message), Branch, Trigger (push/manual), Status, Duration, Deployed by, Timestamp, Actions |
| Filters | Status, branch, date range |
| Row actions | View Logs, Redeploy, Rollback to this build, Compare env vars |
| Empty state | "No deployments yet" |
| Pagination | Cursor-based, 20/page |
| Permissions | Viewer: read-only; Admin+: Rollback |

### 4.7 Deployment Logs (Live)

| Aspect | Detail |
|---|---|
| Purpose | Real-time build/runtime log tailing |
| Components | Log viewer (monospace, auto-scroll toggle), phase timeline (Queued → Cloning → Installing → Building → Deploying → Live), search-within-logs, download button |
| Streaming | Socket.IO channel per deployment ID; buffers last N lines on reconnect |
| Loading | "Connecting to log stream…" state |
| Error state | "Log stream disconnected — reconnecting" with automatic retry |
| Empty state | "No logs yet — waiting for build to start" |
| Permissions | Viewer: read-only |

### 4.8 Environment Variables

| Aspect | Detail |
|---|---|
| Purpose | Manage config/secrets per environment |
| Components | Environment tabs (Production/Staging/Preview), key-value table, "Add Variable", bulk paste modal, reveal/hide toggle per row |
| Forms | Key (validated), value (masked textarea for multiline), "mark as secret" checkbox |
| Modals | Bulk import `.env` paste with diff preview before save |
| Error state | Duplicate key warning, invalid key format inline error |
| Permissions | Admin+ only (secrets are sensitive) |

### 4.9 Domain Management

| Aspect | Detail |
|---|---|
| Purpose | Add/verify custom domains, monitor SSL |
| Components | Domain list table (domain, status badge: Pending/Verifying/Active/Failed, SSL expiry date), "Add Domain" modal with DNS instructions |
| Modals | Add Domain (shows required CNAME/A record to copy), Remove Domain (confirm) |
| Status badges | Pending DNS, Verifying, Active (green), SSL Renewal Failed (red) |
| Empty state | "No custom domains — using default deployhub.app subdomain" |
| Permissions | Admin+ |

### 4.10 Organization Settings

| Aspect | Detail |
|---|---|
| Purpose | Manage org identity, members, roles |
| Components | Org profile form (name, logo), Members table (avatar, name, role dropdown, last active, remove), Invite modal, pending invites list |
| Table actions | Change role (Owner only), Remove member (Admin+) |
| Danger zone | Transfer ownership, delete organization (type-to-confirm) |
| Permissions | Members tab visible to all; edit actions Admin+/Owner only |

### 4.11 Billing (Future — scaffolded now)

| Aspect | Detail |
|---|---|
| Purpose | Plan management, usage, invoices |
| Components | Current plan card, usage bars (build minutes, bandwidth, compute hours) vs. plan limits, plan comparison table, invoice history table |
| Permissions | Owner only |

### 4.12 Profile

| Aspect | Detail |
|---|---|
| Purpose | Personal account settings |
| Components | Avatar (from GitHub), name, connected GitHub identity, API Tokens section, notification preferences, sessions list ("sign out everywhere") |
| Modals | Create API Token (name, scope, expiry) — token shown once, copy-to-clipboard |
| Permissions | Self only |

### 4.13 Notifications Center

| Aspect | Detail |
|---|---|
| Purpose | In-app feed of deploy events, domain/SSL alerts, invites |
| Components | Bell icon with unread count, dropdown/panel list, mark-all-read, notification settings link |
| Empty state | "You're all caught up" |
| Permissions | Self only |

### 4.14 Admin Panel (Internal Ops)

| Aspect | Detail |
|---|---|
| Purpose | Platform-team-only view of system health, all orgs, feature flags |
| Components | Org list with usage stats, system resource dashboard (embedded Grafana), feature flag toggles, impersonation tool (audit-logged) |
| Permissions | Platform admin role only (not exposed to regular customers) |

---

## 5. Backend Design

**Stack:** Node.js + Express + TypeScript, PostgreSQL + Prisma, Redis + BullMQ, Socket.IO, Docker Engine API, Nginx (config generation + reload), JWT.

Backend is organized as a **modular monolith** for Phase 1–3 (simpler ops, single deploy unit) with clear service boundaries so individual modules (e.g., Build Service, Deployment Service) can be extracted into separate services later if scale demands it — this is the same seam that enables the eventual K8s migration.

| Service | Responsibility | Key Interactions |
|---|---|---|
| **Auth Service** | GitHub OAuth exchange, JWT issuance/refresh, session revocation, API token issuance/validation | Prisma (users, sessions, tokens); Redis (refresh token blocklist) |
| **GitHub Service** | Wraps GitHub REST/GraphQL API: list repos, branches, create/verify webhooks, fetch commit metadata, post commit statuses | GitHub API (via installation/OAuth token); Webhook Service |
| **Webhook Service** | Receives GitHub push/PR webhooks, verifies HMAC signature, enqueues deployment job | Deployment Queue; GitHub Service (signature secret lookup) |
| **Deployment Service** | Orchestrates the deploy lifecycle: create deployment record, enqueue build, track state transitions, trigger traffic cutover, trigger rollback | Build Service, Docker Service, Prisma, Socket.IO (status push) |
| **Build Service** | Executes the build: clone repo (shallow), install deps, run build command, build Docker image, tag with commit SHA, push to internal registry | Docker Service; Log Service (streams build output) |
| **Docker Service** | Wraps Docker Engine API: create/start/stop/remove containers, manage networks/volumes, health-check polling, resource limit enforcement | Docker Engine socket; Health Check Service |
| **Log Service** | Aggregates build + runtime logs, streams via Socket.IO, ships to Loki for retention/search | Loki push API; Socket.IO |
| **Domain Service** | Domain CRUD, DNS verification polling, Nginx vhost config generation | SSL Service; Nginx config reload |
| **SSL Service** | ACME (Let's Encrypt) certificate issuance/renewal via certbot or acme.js, cert storage, expiry monitoring | SSL Queue; Domain Service; Nginx reload |
| **Health Check Service** | Polls configured health endpoint during deploy cutover and continuously post-deploy; feeds restart/alerting decisions | Docker Service (restart), Notification Service (alert) |
| **Monitoring / Analytics Service** | Collects container resource metrics (via `docker stats` / cAdvisor), exposes Prometheus metrics endpoint, computes derived stats (failure rate, avg deploy time) | Prometheus scrape target; Prisma (historical rollups) |
| **Notification Service** | Sends email (deploy status, SSL expiry), Slack/Discord webhooks, writes in-app notification rows | Notification Queue; SMTP provider; Prisma |
| **Scheduler** | Cron-like recurring jobs: SSL renewal check, stale build cleanup, health-check sweeps, metrics rollups | BullMQ repeatable jobs |
| **Admin Service** | Platform-internal endpoints: org listing, impersonation (audited), feature flags | Prisma; Audit Log |

### 5.1 Cross-Cutting: Deployment State Machine

```
QUEUED → CLONING → INSTALLING → BUILDING → PUSHING_IMAGE → DEPLOYING → HEALTH_CHECKING → LIVE
                                                                              │
                                                                              ▼
                                                                          FAILED (any stage can transition here)
                                                                              
Rollback: LIVE(target=previous) skips CLONING..PUSHING_IMAGE, goes straight to DEPLOYING → HEALTH_CHECKING → LIVE
Cancel: any non-terminal state → CANCELLED
```

Every transition is persisted (append-only `deployment_events` table) so the UI timeline and audit trail are reconstructable without re-deriving state from logs.

---

## 6. Database Design (PostgreSQL)

### 6.1 ER Diagram (text form)

```
users ──< organization_members >── organizations
  │                                      │
  │                                      ├──< projects >──┐
  │                                      │                │
  ├──< api_tokens                        ├──< invites     │
  ├──< sessions                          │                │
  └──< audit_logs                        └──< audit_logs  │
                                                            │
projects ──< environments ──< environment_variables        │
   │                                                        │
   ├──< deployments ──< deployment_events                   │
   │        │                                               │
   │        └──< deployment_logs (pointer to Loki, not raw text)
   │                                                         │
   ├──< domains ──< ssl_certificates                         │
   │                                                         │
   └──< health_checks                                        │
                                                              │
github_installations ──< projects (via github_repo_id)
notifications >── users
```

### 6.2 Table Definitions

**users**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK, default gen_random_uuid() |
| github_id | bigint | UNIQUE NOT NULL |
| github_username | text | NOT NULL |
| email | text | UNIQUE |
| avatar_url | text | |
| created_at | timestamptz | NOT NULL DEFAULT now() |
| updated_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `UNIQUE(github_id)`, `UNIQUE(email)`

**organizations**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| name | text | NOT NULL |
| slug | text | UNIQUE NOT NULL |
| owner_id | uuid | FK → users.id NOT NULL |
| plan | text | NOT NULL DEFAULT 'free' CHECK (plan IN ('free','pro','team','enterprise')) |
| created_at | timestamptz | NOT NULL DEFAULT now() |

**organization_members**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| organization_id | uuid | FK → organizations.id ON DELETE CASCADE |
| user_id | uuid | FK → users.id ON DELETE CASCADE |
| role | text | NOT NULL CHECK (role IN ('owner','admin','member','viewer')) |
| invited_by | uuid | FK → users.id |
| created_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `UNIQUE(organization_id, user_id)`

**invites**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| organization_id | uuid | FK → organizations.id ON DELETE CASCADE |
| email | text | NOT NULL |
| role | text | NOT NULL |
| token | text | UNIQUE NOT NULL |
| status | text | NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','accepted','expired','revoked')) |
| expires_at | timestamptz | NOT NULL |
| created_at | timestamptz | NOT NULL DEFAULT now() |

**github_installations**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| organization_id | uuid | FK → organizations.id ON DELETE CASCADE |
| github_installation_id | bigint | UNIQUE NOT NULL |
| access_token_encrypted | text | NOT NULL |
| token_expires_at | timestamptz | |
| created_at | timestamptz | NOT NULL DEFAULT now() |

**projects**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| organization_id | uuid | FK → organizations.id ON DELETE CASCADE |
| name | text | NOT NULL |
| slug | text | NOT NULL |
| repo_full_name | text | NOT NULL — e.g. "org/repo" |
| github_repo_id | bigint | NOT NULL |
| default_branch | text | NOT NULL DEFAULT 'main' |
| root_directory | text | NOT NULL DEFAULT '/' |
| runtime | text | NOT NULL DEFAULT 'nodejs' CHECK (runtime IN ('nodejs','python','go','java','rust','dockerfile','static')) |
| build_command | text | |
| start_command | text | |
| port | integer | NOT NULL DEFAULT 3000 |
| status | text | NOT NULL DEFAULT 'active' CHECK (status IN ('active','archived','deleted')) |
| created_by | uuid | FK → users.id |
| created_at | timestamptz | NOT NULL DEFAULT now() |
| updated_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `UNIQUE(organization_id, slug)`, `INDEX(github_repo_id)`

**environments**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| project_id | uuid | FK → projects.id ON DELETE CASCADE |
| name | text | NOT NULL CHECK (name IN ('production','staging','preview')) |
| created_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `UNIQUE(project_id, name)`

**environment_variables**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| environment_id | uuid | FK → environments.id ON DELETE CASCADE |
| key | text | NOT NULL |
| value_encrypted | text | NOT NULL |
| is_secret | boolean | NOT NULL DEFAULT false |
| created_at | timestamptz | NOT NULL DEFAULT now() |
| updated_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `UNIQUE(environment_id, key)`

**deployments**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| project_id | uuid | FK → projects.id ON DELETE CASCADE |
| environment_id | uuid | FK → environments.id |
| commit_sha | text | NOT NULL |
| commit_message | text | |
| branch | text | NOT NULL |
| trigger_type | text | NOT NULL CHECK (trigger_type IN ('push','manual','rollback','redeploy')) |
| triggered_by | uuid | FK → users.id |
| status | text | NOT NULL DEFAULT 'queued' CHECK (status IN ('queued','cloning','installing','building','pushing_image','deploying','health_checking','live','failed','cancelled')) |
| image_tag | text | — internal registry tag, set once built |
| rollback_of_deployment_id | uuid | FK → deployments.id, nullable |
| started_at | timestamptz | |
| finished_at | timestamptz | |
| created_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `INDEX(project_id, created_at DESC)`, `INDEX(status)`

**deployment_events** (append-only state-machine audit trail)
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| deployment_id | uuid | FK → deployments.id ON DELETE CASCADE |
| from_status | text | |
| to_status | text | NOT NULL |
| message | text | |
| created_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `INDEX(deployment_id, created_at)`

**deployment_logs** (pointer table — raw log bodies live in Loki, not Postgres)
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| deployment_id | uuid | FK → deployments.id ON DELETE CASCADE |
| loki_stream_id | text | NOT NULL |
| phase | text | NOT NULL CHECK (phase IN ('build','runtime')) |
| created_at | timestamptz | NOT NULL DEFAULT now() |

**domains**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| project_id | uuid | FK → projects.id ON DELETE CASCADE |
| hostname | text | UNIQUE NOT NULL |
| status | text | NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','verifying','active','failed')) |
| verification_token | text | NOT NULL |
| created_at | timestamptz | NOT NULL DEFAULT now() |

**ssl_certificates**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| domain_id | uuid | FK → domains.id ON DELETE CASCADE |
| issuer | text | NOT NULL DEFAULT 'letsencrypt' |
| issued_at | timestamptz | NOT NULL |
| expires_at | timestamptz | NOT NULL |
| status | text | NOT NULL CHECK (status IN ('valid','expiring_soon','expired','renewal_failed')) |

Indexes: `INDEX(expires_at)` — for renewal sweep queries

**health_checks**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| project_id | uuid | FK → projects.id ON DELETE CASCADE |
| path | text | NOT NULL DEFAULT '/health' |
| interval_seconds | integer | NOT NULL DEFAULT 30 |
| timeout_seconds | integer | NOT NULL DEFAULT 5 |
| unhealthy_threshold | integer | NOT NULL DEFAULT 3 |

**api_tokens**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK → users.id ON DELETE CASCADE |
| name | text | NOT NULL |
| token_hash | text | UNIQUE NOT NULL — SHA-256 of token, raw shown once |
| scope | text | NOT NULL CHECK (scope IN ('read_only','deploy','full')) |
| last_used_at | timestamptz | |
| expires_at | timestamptz | |
| created_at | timestamptz | NOT NULL DEFAULT now() |

**sessions**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK → users.id ON DELETE CASCADE |
| refresh_token_hash | text | UNIQUE NOT NULL |
| user_agent | text | |
| ip_address | inet | |
| revoked | boolean | NOT NULL DEFAULT false |
| expires_at | timestamptz | NOT NULL |
| created_at | timestamptz | NOT NULL DEFAULT now() |

**notifications**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK → users.id ON DELETE CASCADE |
| type | text | NOT NULL |
| payload | jsonb | NOT NULL |
| read_at | timestamptz | |
| created_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `INDEX(user_id, read_at)`

**audit_logs**
| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| organization_id | uuid | FK → organizations.id ON DELETE CASCADE |
| actor_user_id | uuid | FK → users.id |
| action | text | NOT NULL — e.g. "project.deleted", "member.role_changed" |
| target_type | text | |
| target_id | uuid | |
| metadata | jsonb | |
| created_at | timestamptz | NOT NULL DEFAULT now() |

Indexes: `INDEX(organization_id, created_at DESC)`

### 6.3 Design Notes

- **Secrets never stored in plaintext.** `environment_variables.value_encrypted` and `github_installations.access_token_encrypted` use AES-256-GCM with a key managed by AWS KMS (or local envelope encryption in self-hosted mode); decryption only happens server-side, in-memory, at deploy time.
- **Deployments are immutable + append-only events** — this is what makes rollback safe and audit trails trustworthy; you never mutate a deployment's history, only append new state transitions.
- **Raw logs live in Loki, not Postgres** — Postgres stores only a pointer (stream ID), keeping the relational database small and fast; this also lets log retention policy be independent of deployment record retention.
- **All child tables cascade on parent delete** except audit logs and deployment records, which are retained per compliance/audit requirements even if the parent project is deleted (soft-delete pattern recommended for `projects` via the `status` column rather than hard delete).

---

## 7. REST API Design

Base URL: `/api/v1`. Auth: `Authorization: Bearer <jwt|api_token>` unless noted. All mutating endpoints require CSRF-safe token (see Security §10) when called from browser session auth.

### 7.1 Authentication

| Method | Route | Request | Response | Auth | Status Codes |
|---|---|---|---|---|---|
| GET | `/auth/github` | — | 302 redirect to GitHub OAuth | Public | 302 |
| GET | `/auth/github/callback` | `?code=` | Sets HTTP-only cookies (access+refresh), 302 to dashboard | Public | 302, 400 (invalid code) |
| POST | `/auth/refresh` | Refresh cookie | `{ access_token }` | Refresh cookie | 200, 401 |
| POST | `/auth/logout` | — | `{}` | Session | 200 |
| POST | `/auth/logout-all` | — | `{}` — revokes all sessions | Session | 200 |
| GET | `/auth/me` | — | `{ user }` | Session | 200, 401 |

### 7.2 Organizations

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/orgs` | — | `{ organizations: [] }` | 200 |
| POST | `/orgs` | `{ name }` | `{ organization }` | 201, 400 |
| GET | `/orgs/:orgId` | — | `{ organization }` | 200, 404 |
| PATCH | `/orgs/:orgId` | `{ name?, logo_url? }` | `{ organization }` | 200, 403, 404 |
| DELETE | `/orgs/:orgId` | `{ confirm_name }` | `{}` | 204, 403 |
| GET | `/orgs/:orgId/members` | — | `{ members: [] }` | 200 |
| POST | `/orgs/:orgId/invites` | `{ email, role }` | `{ invite }` | 201, 403, 409 |
| POST | `/invites/:token/accept` | — | `{ organization }` | 200, 404, 410 |
| PATCH | `/orgs/:orgId/members/:userId` | `{ role }` | `{ member }` | 200, 403 |
| DELETE | `/orgs/:orgId/members/:userId` | — | `{}` | 204, 403 |

### 7.3 GitHub Integration

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/github/repos` | `?org_id=&search=` | `{ repos: [] }` | 200 |
| GET | `/github/repos/:repoId/branches` | — | `{ branches: [] }` | 200 |
| POST | `/github/webhook` | GitHub payload + `X-Hub-Signature-256` | `{}` | 200, 401 (bad signature) |

### 7.4 Projects

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/orgs/:orgId/projects` | — | `{ projects: [] }` | 200 |
| POST | `/orgs/:orgId/projects` | `{ repo_full_name, github_repo_id, name, branch, build_command?, start_command?, port? }` | `{ project }` | 201, 400, 409 |
| GET | `/projects/:id` | — | `{ project }` | 200, 404 |
| PATCH | `/projects/:id` | partial project fields | `{ project }` | 200, 403 |
| DELETE | `/projects/:id` | — | `{}` | 204, 403 |
| POST | `/projects/:id/archive` | — | `{ project }` | 200 |

### 7.5 Environment Variables

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/projects/:id/environments/:envId/variables` | — | `{ variables: [] }` (values masked unless scope=full) | 200 |
| POST | `/projects/:id/environments/:envId/variables` | `{ key, value, is_secret }` | `{ variable }` | 201, 400, 409 |
| POST | `/projects/:id/environments/:envId/variables/bulk` | `{ raw_env_text }` | `{ created: [], updated: [] }` | 200, 400 |
| PATCH | `/variables/:varId` | `{ value }` | `{ variable }` | 200, 403 |
| DELETE | `/variables/:varId` | — | `{}` | 204, 403 |

### 7.6 Deployments

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/projects/:id/deployments` | `?status=&branch=&page=` | `{ deployments: [], pagination }` | 200 |
| POST | `/projects/:id/deployments` | `{ commit_sha?, branch? }` — manual deploy | `{ deployment }` | 202, 400, 409 (already queued) |
| GET | `/deployments/:id` | — | `{ deployment, events: [] }` | 200, 404 |
| POST | `/deployments/:id/cancel` | — | `{ deployment }` | 200, 409 (already terminal) |
| POST | `/deployments/:id/redeploy` | — | `{ deployment }` (new record, full rebuild) | 202 |
| POST | `/deployments/:id/rollback` | — | `{ deployment }` (new record, no rebuild) | 202, 400 (target not successful) |

### 7.7 Logs

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/deployments/:id/logs` | `?phase=build\|runtime&from=&to=` | `{ lines: [] }` (historical, from Loki) | 200 |
| WS | `/ws/deployments/:id/logs` | Socket.IO channel | Stream of `{ line, timestamp, phase }` | — |

### 7.8 Domains

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/projects/:id/domains` | — | `{ domains: [] }` | 200 |
| POST | `/projects/:id/domains` | `{ hostname }` | `{ domain, dns_instructions }` | 201, 409 |
| POST | `/domains/:id/verify` | — | `{ domain }` | 200, 400 (DNS not found) |
| DELETE | `/domains/:id` | — | `{}` | 204 |

### 7.9 Metrics & Health

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/projects/:id/metrics` | `?range=1h\|24h\|7d` | `{ cpu: [], memory: [], requests: [] }` | 200 |
| GET | `/projects/:id/health` | — | `{ status, last_check_at }` | 200 |
| GET | `/health` | — (platform liveness) | `{ status: "ok" }` | 200 |
| GET | `/metrics` | — (Prometheus scrape endpoint) | Prometheus text format | 200 (internal only) |

### 7.10 API Tokens

| Method | Route | Request | Response | Status Codes |
|---|---|---|---|---|
| GET | `/tokens` | — | `{ tokens: [] }` (no raw value) | 200 |
| POST | `/tokens` | `{ name, scope, expires_in_days? }` | `{ token: "raw-value-shown-once", ...meta }` | 201 |
| DELETE | `/tokens/:id` | — | `{}` | 204 |

### 7.11 Common Conventions

- **Validation:** all bodies validated with Zod schemas at the route boundary; 400 responses return `{ error: { field, message }[] }`.
- **Pagination:** cursor-based (`?cursor=&limit=`), response includes `{ next_cursor }`.
- **Rate limit headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` on every response.
- **Idempotency:** deploy-triggering POSTs accept optional `Idempotency-Key` header to prevent duplicate builds from retried requests.

---

## 8. Background Jobs (BullMQ on Redis)

### 8.1 Queue Map

| Queue | Producer | Consumer(s) | Concurrency | Priority |
|---|---|---|---|---|
| **build-queue** | Webhook Service, manual deploy API | Build Service worker | 2–4 per host (CPU-bound) | High |
| **deployment-queue** | Deployment Service (post-build, and rollback path) | Docker Service worker (start container, cutover) | 4 per host | High |
| **ssl-queue** | Domain Service (on verify success), Scheduler (renewal sweep) | SSL Service worker | 2 | Medium |
| **health-check-queue** | Scheduler (repeatable, every 30s per project) | Health Check Service worker | 10 (I/O-bound) | Medium |
| **notification-queue** | All services (on state change) | Notification Service worker | 5 | Low |
| **metrics-queue** | Scheduler (repeatable, every 60s) | Monitoring Service worker (scrape + rollup) | 3 | Low |
| **cleanup-queue** | Scheduler (daily) | Cleanup worker (old images, stopped containers, expired sessions) | 1 | Low |

### 8.2 Job Definitions

**build-queue → `build:run`**
```
payload: { deploymentId, projectId, commitSha, repoUrl, branch }
steps: clone → install → build → docker build → push to internal registry
on success: enqueue deployment-queue "deploy:cutover"
on failure: mark deployment FAILED, enqueue notification-queue
```

**deployment-queue → `deploy:cutover`**
```
payload: { deploymentId, imageTag, projectId }
steps: start new container → poll health check → switch Nginx upstream → 
       drain + stop old container (30s grace period)
on failure: keep old container live, mark deployment FAILED, alert
```

**deployment-queue → `deploy:rollback`**
```
payload: { deploymentId, targetDeploymentId }
steps: reuse targetDeployment.imageTag → start container → health check → 
       cutover (same as above, skips build entirely)
```

**ssl-queue → `ssl:issue`** / **`ssl:renew`**
```
payload: { domainId }
steps: ACME HTTP-01 challenge → cert issuance → write to volume → 
       reload Nginx → update ssl_certificates row
```

**health-check-queue → `health:check`**
```
payload: { projectId }
repeatable: every 30s (configurable per project)
steps: HTTP GET to health path → record result → on N consecutive failures,
       trigger restart (Docker Service) + notification
```

**notification-queue → `notify:send`**
```
payload: { userId, type, payload }
routes to: email (SMTP), Slack/Discord webhook, in-app notification row
```

**metrics-queue → `metrics:collect`**
```
repeatable: every 60s
steps: docker stats per running container → push to Prometheus pushgateway 
       (or scraped directly, see §11) → rollup hourly aggregates to Postgres
```

**cleanup-queue → `cleanup:sweep`**
```
repeatable: daily at 03:00
steps: remove Docker images older than N successful deploys back,
       remove stopped/orphaned containers, purge expired sessions/invites,
       purge logs past retention window (coordinated with Loki retention config)
```

### 8.3 Retry Strategy & Dead Letter Queue

| Setting | Value | Rationale |
|---|---|---|
| Default retry attempts | 3 | Enough to survive transient network/Docker daemon blips |
| Backoff | Exponential, base 5s (5s, 25s, 125s) | Avoids hammering GitHub API / Docker daemon during outages |
| Build jobs retry | **0 automatic retries** | A failed build is a code/config problem, not transient — auto-retry would confuse users; user must explicitly redeploy |
| Health-check jobs | No retry (it's a poll — next scheduled run is the "retry") | Repeatable job pattern makes explicit retry redundant |
| SSL issuance | 3 retries, longer backoff (1m, 5m, 15m) | ACME rate limits are strict; back off generously |
| Dead Letter Queue | Failed jobs after max attempts moved to `*-dlq` queue, visible in Admin Panel | Prevents silent failure; ops can inspect/replay manually |
| Stalled job detection | BullMQ `lockDuration` + `stalledInterval` tuned per queue (build jobs get longer lock, e.g. 10 min, since builds are long-running) | Prevents a crashed worker from leaving a job stuck "in progress" forever |

---

## 9. Docker Architecture (+ Kubernetes Migration Path)

### 9.1 Phase 1–3: Docker Compose on a Single EC2 Host

**Why not Kubernetes on day one:** K8s adds real operational overhead (control plane, etcd, networking plugins) that isn't justified before there's real multi-tenant load. A single well-provisioned EC2 host running Docker Compose gets to market faster and is dramatically cheaper to operate for the first cohort of customers. The design below is intentionally structured so this isn't a dead end.

**Images**
- One base image per supported runtime (`deployhub/node-builder:20`, etc.) used as the build-stage image; final runtime image is a minimal multi-stage build (e.g. `node:20-slim`) containing only the built app + production deps.
- Every deployed app image is tagged `registry.internal/{projectId}:{commitSha}` — immutable, content-addressable, which is what makes rollback instant (image already exists, just re-run it).

**Containers**
- One container per running deployment. Naming convention: `deploy-{projectId}-{deploymentId}` — deterministic and greppable for debugging via `docker ps`.
- Old container is kept **stopped but not removed** for a grace window (default 24h) after a successful cutover, so an emergency rollback doesn't even need the image pull step — it's already resident.

**Networks**
- Single Docker bridge network (`deployhub-net`) shared by all app containers on the host; each container gets a stable internal hostname so Nginx can proxy by container name rather than dynamic IP.
- Platform's own services (API, workers, Postgres, Redis) run on a **separate internal network**, not reachable from tenant app containers — this is the primary tenant-isolation boundary in Phase 1–3.

**Volumes**
- Named volume per project for any persistent local storage app authors declare (ephemeral by default — DeployHub is stateless-app-first; persistent volumes are opt-in and explicitly flagged as "not portable across future migration to K8s/managed storage").
- Shared volume for Let's Encrypt certs, mounted read-only into the Nginx container.

**Reverse Proxy (Nginx)**
- Single Nginx instance in front of everything; **config is generated, not hand-edited** — Domain Service renders a vhost template per project/domain and triggers `nginx -s reload` (graceful, zero-downtime reload).
- Port mapping: only Nginx binds 80/443 on the host; all app containers bind to ephemeral internal ports only, never exposed to the host directly. This means port collisions between tenant apps are structurally impossible.

**Deployment (Traffic Cutover) Strategy**
1. New container built and started, bound to a fresh internal port, **not yet in the Nginx upstream**.
2. Health Check Service polls the new container directly (bypassing Nginx) until it passes.
3. Nginx config regenerated to point the project's upstream at the new container; graceful reload (existing in-flight connections to the old container are not dropped).
4. Old container removed from upstream, given a drain period, then stopped (not deleted — see grace window above).

This is a **blue-green-per-deployment** pattern — the same conceptual model Kubernetes rolling updates use, which is precisely what makes the migration path below tractable.

**Image Cleanup**
- Cleanup queue (§8) removes images older than the last 5 successful deployments per project, or older than 30 days, whichever is more aggressive — configurable per plan tier.

### 9.2 K8s-Ready Design Seams (built now, activated in Phase 5)

The following decisions in Phase 1–3 are made specifically so that swapping the orchestration layer later doesn't require redesigning the product:

| Seam | Phase 1–3 Implementation | Phase 5 K8s Equivalent |
|---|---|---|
| Unit of deployment | One container per deployment, immutable image | One Pod (or Deployment object) per deployment, same image |
| Traffic cutover | Nginx upstream swap after health check | Kubernetes Service + rolling update, same health-check contract (readiness probe) |
| Health check contract | HTTP GET to configurable path, defined in `health_checks` table | Same config maps directly to `livenessProbe`/`readinessProbe` |
| Config/secrets | `environment_variables` table, injected as container env at start | Same rows map directly to ConfigMap/Secret objects |
| Scaling | Manual container count (Phase 1–3: effectively 1) | HorizontalPodAutoscaler, same CPU/RPS metrics already collected via Prometheus |
| Rollback | Reuse prior image tag, re-run cutover | `kubectl rollout undo` — same immutable-image precondition, same event model |
| Networking isolation | Separate Docker networks for platform vs. tenant | NetworkPolicy objects, same trust boundary |
| Registry | Internal registry (self-hosted or ECR) | Unchanged — same registry, same tags |

Because the **Deployment Service never talks to Docker directly** — it talks to an internal `ContainerOrchestrator` interface — Phase 5 work is primarily writing a Kubernetes-backed implementation of that interface (using the K8s API / client-go equivalent) and standing up a cluster, not rearchitecting the product, database, or API surface.

### 9.3 Deployment Strategy Comparison

| | Phase 1–3 (Docker Compose) | Phase 5+ (Kubernetes) |
|---|---|---|
| Orchestration unit | Container | Pod / Deployment |
| Multi-host | No (single EC2 host) | Yes (multi-node cluster) |
| Autoscaling | Manual only | HPA-based |
| Rollback mechanism | Re-run cutover with old image | `kubectl rollout undo` (or equivalent API call) |
| Isolation | Docker network segmentation | Namespaces + NetworkPolicy |
| Operational complexity | Low | Higher, justified by scale |

---

## 10. Security

| Area | Approach |
|---|---|
| **JWT** | Short-lived access tokens (15 min), signed RS256, stored in HTTP-only + Secure + SameSite=Lax cookie (not localStorage, to mitigate XSS token theft) |
| **Refresh Tokens** | Long-lived (30 days), rotated on every use (old one invalidated immediately — reuse of a rotated token revokes the entire session family, a signal of theft) |
| **RBAC** | Roles enforced at the middleware layer per route (`requireRole('admin')`), never inferred from client-supplied data; role checks re-validated server-side even if UI already hid the action |
| **Secrets Encryption** | AES-256-GCM envelope encryption; data-encryption-keys wrapped by a KMS-managed key (AWS KMS in hosted mode, local master key + rotation docs in self-hosted mode) |
| **Rate Limiting** | Redis-backed sliding window; stricter limits on auth endpoints (`/auth/*`: 10/min/IP) and deploy-triggering endpoints (`/deployments`: 20/hour/project) than read endpoints |
| **CORS** | Allowlist of known frontend origins only; credentials mode restricted to those origins; no wildcard `*` with credentials |
| **Helmet** | Standard secure headers (CSP, HSTS, X-Frame-Options DENY, X-Content-Type-Options nosniff) applied globally |
| **CSRF** | Double-submit cookie token for state-changing requests made with cookie-based session auth; not required for Bearer-token API-client requests (no ambient cookie to forge) |
| **Docker Isolation** | Tenant containers run as non-root user inside the image; `--cap-drop=ALL` with only required capabilities re-added; read-only root filesystem where the app allows it; resource limits (`--memory`, `--cpus`) enforced per plan tier to prevent noisy-neighbor issues on the shared host |
| **Container Security** | Base images pulled only from pinned, scanned tags; dependency/image scanning (e.g. Trivy) integrated into the internal build pipeline before an image is eligible to run |
| **Input Validation** | Zod schemas at every API boundary; reject unknown fields; strict typing on env var keys to prevent shell-injection-style surprises when values are later interpolated into container start commands |
| **Audit Logs** | Every mutating action on org/project/member/domain writes an `audit_logs` row with actor, action, target, and metadata — append-only, not user-deletable |
| **API Security** | All tokens scoped (read_only / deploy / full); tokens hashed at rest (never store raw); last-used timestamp tracked to help users spot stale/compromised tokens |
| **GitHub Webhook Verification** | Every incoming webhook's `X-Hub-Signature-256` HMAC verified against the per-installation shared secret before the payload is trusted or enqueued; requests failing verification are rejected with 401 and logged, never processed |

---

## 11. Monitoring

| Layer | Tool | What's Collected |
|---|---|---|
| **Metrics** | Prometheus | Container CPU/memory/disk per deployment, HTTP request count/latency (via Nginx `stub_status`/log-based exporter), queue depth per BullMQ queue, deployment duration, deployment failure rate |
| **Dashboards** | Grafana | Per-project dashboard (embedded in Project Overview page), platform-wide dashboard (Admin Panel): host resource usage, active deployments, queue health |
| **Logs** | Loki + Promtail | Build logs, runtime stdout/stderr per container, Nginx access/error logs, backend service logs — all labeled by `project_id`/`deployment_id` for fast filtering |
| **Alerting** | Alertmanager (Prometheus stack) | Rules: container down > 1 min, health check failing 3x consecutive, disk usage > 85%, SSL cert expiring < 14 days, queue backlog > threshold, deploy failure rate > 20% in 1h |
| **Alert delivery** | Notification Service | Routes Alertmanager webhooks → email / Slack / Discord, and creates in-app notifications |

**Deployment Time & Failure Rate** are computed both as real-time Prometheus metrics (for alerting) and as historical Postgres rollups (for the Deployment History UI and long-term trend charts) — this avoids querying Prometheus's shorter retention window for data the product wants to show for months.

---

## 12. Project Folder Structure

### 12.1 Backend

```
backend/
├── src/
│   ├── modules/
│   │   ├── auth/            # controllers, service, dto, guards
│   │   ├── github/
│   │   ├── projects/
│   │   ├── deployments/
│   │   ├── docker/           # ContainerOrchestrator interface + Docker impl
│   │   ├── build/
│   │   ├── domains/
│   │   ├── ssl/
│   │   ├── health/
│   │   ├── monitoring/
│   │   ├── notifications/
│   │   ├── organizations/
│   │   └── admin/
│   ├── queues/                # BullMQ queue + worker definitions, one file per queue
│   ├── websocket/             # Socket.IO gateway (log streaming, deploy status push)
│   ├── prisma/                # schema.prisma, migrations/
│   ├── middleware/            # auth guard, rbac guard, rate-limit, error handler
│   ├── lib/                   # crypto (encryption helpers), logger, config loader
│   ├── jobs/                  # scheduler (repeatable job registration)
│   └── app.ts / server.ts
├── prisma/schema.prisma
├── docker/                    # Dockerfiles for platform services themselves
├── tests/
└── package.json
```

*Rationale: feature-modules (not layer-first folders) keep everything about "deployments" in one place — easier to eventually extract as its own service, and mirrors the Backend Design service list in §5 one-to-one.*

### 12.2 Frontend

```
frontend/
├── app/                        # Next.js App Router
│   ├── (marketing)/landing/
│   ├── (auth)/login/
│   ├── (dashboard)/
│   │   ├── dashboard/
│   │   ├── projects/[projectId]/
│   │   │   ├── overview/
│   │   │   ├── deployments/
│   │   │   ├── logs/
│   │   │   ├── env-vars/
│   │   │   ├── domains/
│   │   │   └── settings/
│   │   ├── org/[orgId]/settings/
│   │   ├── profile/
│   │   └── admin/
│   └── api/                    # Next.js route handlers if any BFF endpoints needed
├── components/
│   ├── ui/                     # shadcn/ui primitives
│   ├── shared/                 # cross-page components (StatusBadge, LogViewer, etc.)
│   └── charts/
├── hooks/                      # React Query hooks per resource (useProjects, useDeployments)
├── stores/                     # Zustand stores (UI state: sidebar, modals)
├── lib/                        # API client, auth helpers, formatters
├── types/                      # shared TS types (mirrors backend DTOs)
└── styles/
```

*Rationale: route-colocated feature folders under `app/` mirror the Frontend Design page list in §4 one-to-one; `hooks/` centralizes React Query so caching/invalidation logic isn't duplicated across pages.*

---

## 13. Development Roadmap

| Phase | Scope | Key Deliverables | Est. Effort | Dependencies |
|---|---|---|---|---|
| **Phase 1 — Foundation** | Auth, projects, basic deploy (no rollback yet) | GitHub OAuth, JWT, project import, Node.js build pipeline, single-container deploy, Nginx cutover, basic logs (streamed, not persisted) | 5–6 weeks | GitHub OAuth App approval, EC2 + Docker host provisioned |
| **Phase 2 — Reliability** | Rollback, health checks, env vars, domains/SSL | Immutable deployment history, rollback flow, health-check-gated cutover, encrypted env vars, custom domains + Let's Encrypt automation | 5–6 weeks | Phase 1 complete |
| **Phase 3 — Team & Ops** | Orgs/teams/RBAC, notifications, monitoring, API tokens | Multi-user orgs, roles, invites, Prometheus/Grafana/Loki stack, email + Slack notifications, API tokens, audit logs | 5–7 weeks | Phase 2 complete |
| **Phase 4 — Multi-Runtime** | Python/Go/Java/Rust/Dockerfile/static support | Buildpack abstraction layer, per-runtime builder images, static-site CDN-style serving | 6–8 weeks | Phase 3 complete; build pipeline generalized |
| **Phase 5 — Scale-Out** | Kubernetes orchestration option, autoscaling, multi-region | `ContainerOrchestrator` K8s implementation (per §9.2), HPA, multi-node cluster, region selection at project creation | 8–10 weeks | Phase 4 complete; K8s-ready seams from Phase 1–3 pay off here |
| **Phase 6 — Monetization** | Billing, usage metering, marketplace add-ons | Stripe integration, plan enforcement, usage dashboards, PR preview environments, add-on databases/Redis provisioning | 6–8 weeks | Phase 3 (orgs) and Phase 5 (scaling) complete |
| **Phase 7 — Enterprise / Self-Hosted** | BYO-cloud deployment, SSO, compliance | Helm chart / Terraform module for self-hosted control plane, SAML/OIDC SSO, SCIM, enhanced audit export | 8–12 weeks | Phase 5 (K8s) and Phase 6 (billing model for Enterprise tier) complete |

**Sequencing rationale:** Rollback and health checks (Phase 2) are pulled forward ahead of team features because a deploy platform without a reliable undo button is not credible to any user, even a solo developer — this is the platform's core trust promise. Multi-runtime support (Phase 4) is deliberately after team/ops maturity (Phase 3) because expanding the runtime matrix multiplies QA surface area, and it's cheaper to do that expansion once the observability stack already exists to catch regressions. Kubernetes (Phase 5) is sequenced last among the technical phases specifically because Phases 1–3 were designed with the seams in §9.2 — pulling it forward earlier would be premature optimization for load that doesn't exist yet.
