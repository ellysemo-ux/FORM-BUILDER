# FormFlow: Client Intake & Requirements Platform

> A multi-tenant platform where businesses build **customized forms** to collect **customer details, documents, and project requirements**, then manage submissions through a review pipeline.

[![CI](https://github.com/YOUR_ORG/formflow/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR_ORG/formflow/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## 1. Vision and Scope

**Problem:** Businesses collect client info over email, WhatsApp, and scattered PDFs. Requirements get lost, documents arrive in wrong formats, and follow-ups are manual.

**Solution:** One branded link per form. Customers fill it in, upload documents, and the business gets structured, searchable, trackable submissions.

**Primary users**
| Role | Needs |
|---|---|
| Business Owner / Admin | Build forms, manage team, view all submissions, export data |
| Team Member | Review submissions, comment, change status |
| Customer (public, no account) | Fill form, upload files, save progress, get confirmation |

---

## 2. Core Features

### MVP (build first)
- [ ] Authentication, organizations (workspaces), team invites
- [ ] **Drag-and-drop form builder** (text, email, phone, number, date, dropdown, checkbox, radio, textarea, file upload, signature)
- [ ] Public shareable form link and embeddable widget
- [ ] Secure document upload (type/size limits, virus scan)
- [ ] Submission inbox with search, filters, status
- [ ] Email notifications (to business and confirmation to customer)
- [ ] CSV export

### Phase 2
- [ ] Conditional logic (show/hide fields based on answers)
- [ ] Multi-step forms with progress bar and **save & resume**
- [ ] Form templates (client onboarding, web project brief, legal intake, etc.)
- [ ] Custom branding (logo, colors, custom domain)
- [ ] Webhooks and Zapier/Make integration
- [ ] Comments, assignments, and status pipeline (New, In Review, Needs Info, Approved)
- [ ] E-signature and PDF generation of submissions

### Phase 3
- [ ] Payments for deposits (Stripe / M-Pesa via Daraja or Flutterwave)
- [ ] AI summary of project requirements, auto-tagging, missing-info detection
- [ ] Client portal (customers track status, upload more files)
- [ ] Analytics (completion rate, drop-off per field)
- [ ] Multi-language forms, WhatsApp/SMS notifications
- [ ] Public API + API keys

---

## 3. Recommended Tech Stack

| Layer | Recommendation | Why |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript** | One codebase for UI and API, great DX, easy hosting |
| UI | Tailwind CSS + shadcn/ui | Fast, accessible, consistent |
| Form builder | `dnd-kit` + **React Hook Form** + **Zod** | Drag-drop, validation shared client/server |
| Database | **PostgreSQL** (Neon, Supabase, or RDS) | Relational + `JSONB` for flexible form schemas |
| ORM | **Prisma** or **Drizzle** | Type-safe migrations |
| Auth | **Auth.js** or Clerk / Supabase Auth | Don't build your own auth |
| File storage | **S3-compatible** (AWS S3, Cloudflare R2, Backblaze) | Cheap, scalable, presigned uploads |
| Virus scanning | **ClamAV** (self-hosted) or a scanning API | Uploaded docs are an attack vector |
| Background jobs | **BullMQ + Redis** or Inngest/Trigger.dev | Emails, scans, webhooks, PDF generation |
| Email | Resend, Postmark, or AWS SES | Deliverability |
| PDF | Puppeteer / `@react-pdf/renderer` | Submission exports |
| Search | Postgres full-text first, Meilisearch later | Avoid early complexity |
| Monitoring | Sentry + OpenTelemetry + Better Stack/Grafana | Catch errors before customers do |
| Testing | Vitest (unit), Playwright (E2E) | Fast and reliable |
| Infra | Docker, Vercel/Fly.io/Railway to start, Terraform later | Low ops at the beginning |

> **Alternative stack:** NestJS (API) + React (Vite) + PostgreSQL if you want a strict separation of frontend and backend or a larger team.

---

## 4. Architecture

```
┌────────────┐   ┌────────────────────────┐   ┌──────────────┐
│ Form Builder│  │  Public Form Renderer   │   │ Admin Inbox  │
│  (Admin)    │  │  (Customers, no login)  │   │  (Team)      │
└─────┬──────┘   └───────────┬────────────┘   └──────┬───────┘
      │                      │                        │
      └──────────────┬───────┴────────────────────────┘
                     ▼
            ┌─────────────────┐
            │  API / Server   │── Auth, RBAC, validation (Zod)
            │  Actions        │── Rate limiting, CSRF, audit log
            └───┬─────┬───┬───┘
                │     │   │
      ┌─────────▼┐ ┌──▼───┐ ┌▼────────────┐
      │PostgreSQL│ │Redis │ │ S3 / R2      │
      │ (JSONB)  │ │queue │ │ (presigned)  │
      └──────────┘ └──┬───┘ └──────┬──────┘
                      ▼            ▼
              Workers: email, webhooks,
              virus scan, PDF, AI summary
```

**Key principle:** A form is **data, not code**. Store form definitions as versioned JSON schema and render them dynamically.

---

## 5. Data Model (simplified)

```sql
organizations   (id, name, slug, plan, branding_json, created_at)
users           (id, email, name, password_hash NULL, created_at)
memberships     (id, org_id, user_id, role)            -- owner | admin | member | viewer

forms           (id, org_id, title, slug, status, current_version_id)
form_versions   (id, form_id, version, schema_json, settings_json, created_at)  -- immutable

submissions     (id, org_id, form_id, form_version_id, status, data_json,
                 customer_email, customer_name, submitted_at, assigned_to)
files           (id, org_id, submission_id, field_key, storage_key, original_name,
                 mime_type, size_bytes, scan_status, created_at)
comments        (id, submission_id, user_id, body, created_at)
audit_logs      (id, org_id, actor_id, action, entity, entity_id, meta_json, ip, created_at)
webhooks        (id, org_id, url, secret, events[], active)
api_keys        (id, org_id, hashed_key, scopes[], last_used_at)
```

**Why versioning matters:** If a business edits a form after 200 submissions, old submissions must still render against the schema they were submitted with. Always link `submissions.form_version_id`.

### Example form schema (stored as JSONB)

```json
{
  "title": "Website Project Brief",
  "steps": [
    {
      "id": "contact",
      "title": "Your Details",
      "fields": [
        { "key": "full_name", "type": "text", "label": "Full name", "required": true },
        { "key": "email", "type": "email", "label": "Email", "required": true },
        { "key": "phone", "type": "phone", "label": "Phone" }
      ]
    },
    {
      "id": "project",
      "title": "Project Requirements",
      "fields": [
        {
          "key": "project_type", "type": "select", "label": "Project type",
          "options": ["Website", "Mobile app", "Branding", "Other"], "required": true
        },
        {
          "key": "other_type", "type": "text", "label": "Please specify",
          "visibleIf": { "field": "project_type", "equals": "Other" }
        },
        { "key": "budget", "type": "select", "label": "Budget range",
          "options": ["< $1k", "$1k-$5k", "$5k-$20k", "$20k+"] },
        { "key": "brief", "type": "textarea", "label": "Describe your project", "maxLength": 5000 }
      ]
    },
    {
      "id": "docs",
      "title": "Documents",
      "fields": [
        {
          "key": "attachments", "type": "file", "label": "Upload documents",
          "accept": ["application/pdf", "image/png", "image/jpeg", ".docx"],
          "maxSizeMB": 25, "maxFiles": 5
        }
      ]
    }
  ]
}
```

---

## 6. Best Practices: Forms and Data Collection

1. **Server-side validation always.** Generate a Zod schema from the form JSON and validate on the server. Never trust the browser.
2. **Save & resume.** Auto-save drafts (debounced) and give a magic-link to continue later. Long forms lose customers.
3. **Multi-step with progress.** Break long forms into 3 to 7 fields per step.
4. **Conditional logic engine.** Keep rules declarative (`visibleIf`, `requiredIf`) and evaluate them identically on client and server.
5. **Idempotent submissions.** Use an idempotency key to prevent duplicate submissions on double-click or retry.
6. **Spam protection.** Honeypot field + Cloudflare Turnstile/hCaptcha + rate limiting per IP.
7. **Accessible by default.** WCAG 2.2 AA: labels, keyboard nav, focus states, error messages linked via `aria-describedby`.
8. **Mobile first.** Most customers will open the link on a phone. Test on low-bandwidth connections.
9. **Confirmation + reference number.** Show a thank-you page and email a copy of the submission.
10. **Field-level encryption** for highly sensitive fields (IDs, tax numbers), optional per field.

---

## 7. Best Practices: Document Uploads (high risk area)

- ✅ **Direct-to-storage uploads** using presigned URLs so large files never hit your server.
- ✅ Validate **extension, MIME type, and magic bytes** (not just the filename).
- ✅ Enforce **size and count limits** per field and per organization plan.
- ✅ **Virus scan** every file; keep `scan_status = pending | clean | infected` and block download until `clean`.
- ✅ Store files in a **private bucket**; serve through **short-lived signed URLs**.
- ✅ **Randomize storage keys** (`org/{orgId}/sub/{subId}/{uuid}`), never trust the original filename.
- ✅ Strip EXIF metadata from images; sanitize PDFs if previewing.
- ✅ Support **chunked/resumable uploads** (tus protocol or S3 multipart) for big files.
- ✅ Set **retention policies** and lifecycle rules (auto-delete after N days if requested).
- ❌ Never render uploaded HTML/SVG inline. Serve with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`.

---

## 8. Security Checklist

| Area | Recommendation |
|---|---|
| **Multi-tenancy** | Every query scoped by `org_id`. Use **Postgres Row-Level Security** as a second safety net |
| **AuthZ (RBAC)** | owner / admin / member / viewer; check permissions in a central policy layer |
| **Auth** | MFA for admins, secure session cookies (`HttpOnly`, `Secure`, `SameSite`), rate-limited login |
| **Input** | Validate everything with Zod; parameterized queries only; sanitize any rich text |
| **Transport** | HTTPS everywhere, HSTS, strict CSP, secure headers (helmet-style) |
| **Secrets** | Environment variables / secret manager; never commit `.env`; enable GitHub secret scanning |
| **Webhooks** | Sign payloads with HMAC; include timestamp; retry with exponential backoff |
| **API keys** | Store only hashes; show once; support scopes and rotation |
| **Audit logging** | Log who viewed, exported, deleted, or changed what |
| **Backups** | Automated daily DB backups + point-in-time recovery; test restores |
| **Dependencies** | Dependabot + `npm audit` in CI + lockfile committed |
| **Pen-testing** | OWASP Top 10 review before public launch |

---

## 9. Privacy and Compliance

You are handling **personal data and client documents**. Build compliance in early:

- Publish a **Privacy Policy, Terms of Service, and Data Processing Agreement (DPA)**.
- Support **GDPR/CCPA-style rights**: export and delete a customer's data on request. Also review the data protection law of each country where your customers operate.
- **Consent checkbox** and purpose text on forms that collect personal data.
- **Data residency** option (choose storage region) as a paid or enterprise feature.
- **Retention settings** per form (auto-delete submissions after X months).
- Consider **SOC 2 / ISO 27001** once you target larger customers.
- Avoid collecting more than you need (data minimization).

---

## 10. Suggested Repository Structure

```
formflow/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                 # lint, typecheck, test, build
│   │   ├── e2e.yml                # Playwright
│   │   └── codeql.yml             # security scanning
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.yml
│   │   └── feature_request.yml
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── CODEOWNERS
│   └── dependabot.yml
├── apps/
│   ├── web/                       # Next.js app (builder, renderer, dashboard)
│   └── worker/                    # background jobs
├── packages/
│   ├── db/                        # Prisma/Drizzle schema + migrations
│   ├── form-engine/               # schema types, validation, conditional logic
│   ├── ui/                        # shared components
│   └── config/                    # eslint, tsconfig, tailwind presets
├── docs/
│   ├── architecture.md
│   ├── api.md
│   └── adr/                       # Architecture Decision Records
├── docker-compose.yml             # Postgres, Redis, MinIO, ClamAV, Mailpit
├── .env.example
├── CONTRIBUTING.md
├── SECURITY.md
├── CODE_OF_CONDUCT.md
├── LICENSE
└── README.md
```

> Use a **monorepo** (pnpm workspaces + Turborepo). The `form-engine` package is shared by the builder, the public renderer, and the server for consistent validation.

---

## 11. Local Development Setup

```bash
# 1. Clone
git clone https://github.com/YOUR_ORG/formflow.git && cd formflow

# 2. Install
pnpm install

# 3. Environment
cp .env.example .env

# 4. Start services (Postgres, Redis, MinIO for S3, ClamAV, Mailpit for emails)
docker compose up -d

# 5. Database
pnpm db:migrate && pnpm db:seed

# 6. Run
pnpm dev          # web on http://localhost:3000
pnpm dev:worker   # background jobs
```

### `.env.example`
```bash
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/formflow
REDIS_URL=redis://localhost:6379

AUTH_SECRET=change-me
APP_URL=http://localhost:3000

S3_ENDPOINT=http://localhost:9000
S3_REGION=auto
S3_BUCKET=formflow-uploads
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin

CLAMAV_HOST=localhost
CLAMAV_PORT=3310

EMAIL_FROM="FormFlow <no-reply@example.com>"
RESEND_API_KEY=

TURNSTILE_SITE_KEY=
TURNSTILE_SECRET_KEY=

SENTRY_DSN=
```

---

## 12. API Design (REST, versioned)

```
POST   /api/v1/forms                       Create form
GET    /api/v1/forms/:id                   Get form
PUT    /api/v1/forms/:id                   Update (creates new version on publish)
POST   /api/v1/forms/:id/publish           Publish current draft

GET    /public/forms/:slug                 Public form schema
POST   /public/forms/:slug/drafts          Save progress
POST   /public/forms/:slug/uploads/sign    Get presigned upload URL
POST   /public/forms/:slug/submit          Submit (rate-limited, idempotent)

GET    /api/v1/submissions                 List (filter, search, paginate)
GET    /api/v1/submissions/:id             Detail
PATCH  /api/v1/submissions/:id             Status / assignee
GET    /api/v1/submissions/:id/files/:fid  Signed download URL
POST   /api/v1/submissions/:id/comments    Add comment
GET    /api/v1/submissions/export.csv      Export
```

Recommendations: cursor pagination, consistent error format (RFC 7807), OpenAPI spec generated from Zod, and API keys with scopes for integrations.

---

## 13. Performance and Scalability

- Cache published form schemas (CDN/edge) since they are read-heavy and change rarely.
- Index `submissions(org_id, form_id, submitted_at DESC)` and GIN index on `data_json` fields you filter on.
- Paginate everything; never load all submissions.
- Process emails, scans, and webhooks **asynchronously** in queues with retries and dead-letter handling.
- Use a CDN for the public form pages and static assets.
- Load-test the submit endpoint (k6) before launch.

---

## 14. GitHub Workflow Recommendations

**Branching:** trunk-based with short-lived feature branches (`feat/`, `fix/`, `chore/`).

**Branch protection on `main`:**
- Require PR + 1 approval
- Require passing CI checks
- Require linear history, disallow force-push
- Require signed commits (optional)

**Commit style:** [Conventional Commits](https://www.conventionalcommits.org/) → automated changelog and versioning with `release-please` or `changesets`.

**Enable in repo Settings → Security:**
- Dependabot alerts and updates
- Secret scanning + push protection
- CodeQL code scanning
- Private vulnerability reporting (`SECURITY.md`)

### Minimal `ci.yml`
```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: postgres }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build
```

**Issue and project management:** use GitHub Projects with milestones matching the roadmap below.

---

## 15. Testing Strategy

| Type | Tool | What to cover |
|---|---|---|
| Unit | Vitest | Form engine, validation, conditional logic, permissions |
| Integration | Vitest + test DB | API routes, multi-tenant isolation |
| E2E | Playwright | Build form → publish → submit with file → review |
| Security | CodeQL, `npm audit`, ZAP | Injection, auth bypass, upload abuse |
| Accessibility | axe-core | Public form pages |
| Load | k6 | Submission endpoint, uploads |

**Must-have test:** user from Org A can never read, write, or download anything from Org B.

---

## 16. Deployment

| Stage | Option |
|---|---|
| **Start (cheap & fast)** | Vercel (web) + Neon (Postgres) + Cloudflare R2 + Upstash Redis |
| **Container-based** | Fly.io / Railway / Render with Docker |
| **Scale-up** | AWS ECS/EKS or GCP Cloud Run + RDS + S3, infra as code with Terraform |

- Separate **staging** and **production** environments.
- Run DB migrations in CI/CD with a rollback plan.
- Use health checks, uptime monitoring, and alerting.

---

## 17. Business and Monetization Ideas

| Plan | Limits / Features |
|---|---|
| **Free** | 3 forms, 50 submissions/mo, 100 MB storage, branding badge |
| **Pro** | Unlimited forms, 1,000 submissions, 10 GB, custom branding, webhooks |
| **Business** | Team roles, custom domain, audit logs, SSO, priority support |
| **Enterprise** | Data residency, SLA, DPA, dedicated support |

Usage-based add-ons: extra storage, extra submissions, e-signatures, AI summaries.

---

## 18. Roadmap

| Milestone | Goal | Estimate |
|---|---|---|
| **M0: Foundation** | Repo, CI, auth, orgs, DB schema, Docker setup | 1-2 weeks |
| **M1: Builder + Renderer** | Form builder, public form, validation, submissions inbox | 3-4 weeks |
| **M2: Documents** | Secure uploads, virus scan, signed downloads | 2 weeks |
| **M3: Workflow** | Statuses, comments, notifications, CSV export | 2 weeks |
| **M4: Beta launch** | Templates, branding, rate limiting, monitoring, docs | 2-3 weeks |
| **M5: Growth** | Conditional logic, webhooks, integrations, billing | 4+ weeks |
| **M6: Advanced** | Client portal, AI summaries, analytics, public API | ongoing |

---

## 19. Common Pitfalls to Avoid

1. ❌ Building the form builder with hard-coded fields instead of a **JSON schema**
2. ❌ Not versioning forms, which breaks old submissions
3. ❌ Trusting client-side validation only
4. ❌ Uploading files through your own server (slow, costly, risky)
5. ❌ Skipping virus scanning and file type validation
6. ❌ Forgetting `org_id` scoping on one query and leaking tenant data
7. ❌ Over-building (AI, payments, portal) before validating the core with real users
8. ❌ Ignoring mobile experience and slow connections
9. ❌ No audit logs, then having no answer when a client asks "who viewed this?"
10. ❌ Storing everything forever, which creates privacy and cost liabilities

---

## 20. First Steps Checklist

- [ ] Create the GitHub repo, add this README, `LICENSE`, `SECURITY.md`, `CONTRIBUTING.md`
- [ ] Set branch protection and enable Dependabot, secret scanning, CodeQL
- [ ] Scaffold the monorepo (`pnpm` + Turborepo + Next.js + TypeScript)
- [ ] Write `docker-compose.yml` and `.env.example`
- [ ] Define the DB schema and first migration
- [ ] Build the `form-engine` package (types, Zod generator, conditional logic) with tests
- [ ] Implement auth + organization + membership
- [ ] Ship the thinnest end-to-end slice: **create form → share link → submit → see in inbox**
- [ ] Interview 5 to 10 target businesses and test with them before adding Phase 2 features

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Please open an issue before large changes.

## Security

Found a vulnerability? Do **not** open a public issue. See [SECURITY.md](SECURITY.md).

## License

MIT, see [LICENSE](LICENSE).
