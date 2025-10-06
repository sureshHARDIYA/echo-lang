## Echo-Lang LMS — Architecture Plan (Draft)

### Goals
- **Adaptive learning**: Recommend the next best action based on learner profile, mastery, and goals.
- **Free exploration**: Learners can navigate freely; the system still provides guidance.
- **Hybrid content**: AI-generated and human-authored content with rigorous review and safety.
- **Open source and secure by default**: Strong authn/z, data protection, observability, and supply-chain security.
- **Composable services**: Start small, evolve to microservices as needs grow.

### Non‑Goals (initially)
- Payment/billing, marketplace, and extensive gamification (can be added later).
- Full-blown mobile offline support (plan for later phases).

## Core Principles
- **Privacy-first**: Minimize PII, encrypt sensitive data, clear retention policies.
- **Least privilege**: RBAC + ABAC; scoped tokens; service identity and mTLS.
- **Evolutionary architecture**: MVP as a modular monolith, with clean boundaries and event contracts enabling later extraction.
- **Observability**: First-class tracing, metrics, logs; audit trails.
- **Data stewardship**: Versioned content, reproducible AI pipelines, lineage.

## High-Level Architecture

```mermaid
graph TD

  A[Web/App (Next.js)] -->|Requests| B[BFF/API Gateway]
  B -->|OIDC| C[Auth Provider (Keycloak)]

  subgraph Core_Services
    D[User/Profile Service]
    E[Content Service]
    F[Progress/Telemetry Service]
    L[Recommendation/Adaptation]
    M[Assessment/Scoring]
    N[Search]
    O[Notifications]
    P[Policy/AuthZ (OPA)]
  end

  B --> D
  B --> E
  B --> F
  B --> L
  B --> M
  B --> N
  B --> O
  B --> P

  subgraph Eventing
    I[(Event Bus: NATS/Kafka)]
  end

  D -->|events| I
  E -->|events| I
  F -->|events| I
  L -->|events| I
  M -->|events| I

  D --> G[(PostgreSQL OLTP)]
  E --> G
  F --> G
  P -.-> G
  D --> H[(Redis Cache)]
  E --> J[(S3/MinIO Objects)]
  F --> K[(Warehouse: ClickHouse/BigQuery)]
```

Note: For MVP, the BFF and core services can live in a single repository as a modular monolith, using internal module boundaries and an internal event bus. Extract to independent services when scale/teams require.

## Services (boundaries and responsibilities)
- **Web App / BFF**
  - Next.js frontend with server actions or API routes for BFF responsibilities.
  - Handles session, caches, and composes data for the UI.
  - Enforces coarse authZ and delegates fine-grained checks to Policy service.
- **AuthN/Identity**
  - Keycloak (OSS) as the OIDC/OAuth2 provider; supports PKCE, refresh tokens rotation.
  - Social logins optional. Supports service accounts and mTLS between services.
- **User/Profile Service**
  - Stores learner profile, goals, preferences, locales, privacy settings.
  - Tracks roles: learner, author, reviewer, admin.
- **Content Service**
  - Manages human-authored units, lessons, exercises; versioning and approvals.
  - Holds metadata: language, CEFR/ACTFL level, skills, prerequisites, tags.
  - Stores content in Postgres (structured) and S3 (media). Indexes to Search.
- **AI Content Service**
  - Templated prompt workflows, evaluation harness, safety filters, and human-in-the-loop review queues.
  - Stores generations and provenance; enforces licensing and reuse policies.
- **Assessment/Scoring Service**
  - Item bank, rubric, auto-scoring pipelines (LLM-assisted for constructed responses with guardrails).
  - Records attempt data and normalized scores with uncertainty estimates.
- **Progress/Telemetry Service**
  - Writes learning events (xAPI-like) and aggregates progress, mastery estimations.
  - Emits events for recommendation updates. Supports audit queries.
- **Recommendation/Adaptation Service**
  - Maintains learner model (e.g., skills mastery via Bayesian Knowledge Tracing or Elo-like difficulty adjustment).
  - Selects next best activity given goals, constraints, novelty, and diversity.
  - Online learning with contextual bandits; A/B experimentation hooks.
- **Search Service**
  - Meilisearch/OpenSearch index for content discovery and free navigation.
- **Notifications Service**
  - Email/push/in-app messaging; digest generation; rate-limiting and preferences.
- **Policy/Authorization Service**
  - Central ABAC/RBAC via OPA (Rego policies) + entitlements; evaluation exposed via sidecar or API.
  - Fine-grained decisions on content visibility, PII access, and admin actions.

## Data and Storage
- **Primary OLTP**: PostgreSQL (normalized, strong constraints; use Prisma/Drizzle for TS or SQL migrations).
- **Cache**: Redis (sessions, rate limits, hot queries, recommendation candidates).
- **Object Storage**: S3/MinIO for media, exports, model artifacts.
- **Search Index**: Meilisearch (developer-friendly, OSS) or OpenSearch (for larger scale).
- **Event Bus**: NATS (simple) or Kafka (heavier, exactly-once pipelines).
- **Analytics/Warehouse**: ClickHouse (OSS) or DuckDB/Parquet for self-host; BigQuery if managed.

### Minimal Initial Schema (conceptual)
- `users(id, sub, email, locale, roles, created_at)`
- `profiles(user_id, goals, proficiency, interests, preferences_json)`
- `skills(id, name, language, cefr_level, parent_skill_id)`
- `content(id, type, language, skill_id, level, status, version, author_id, metadata_json)`
- `content_versions(content_id, version, body_richtext, rubric_json, created_at)`
- `attempts(id, user_id, content_id, score, confidence, raw_response_json, created_at)`
- `events(id, user_id, type, payload_json, created_at)`
- `progress(user_id, skill_id, mastery_mu, mastery_sigma, last_updated)`

## Adaptive Learning Model (initial approach)
- **Skill graph**: Directed graph over `skills` with prerequisites and difficulty.
- **Learner model**: Mastery as distribution per skill (mu/sigma). Initialize from placement test or prior data; update on attempts.
- **Policy**: Contextual bandit that balances exploitation (highest expected gain) and exploration (reduce uncertainty, diversify).
- **Cold start**: Quick diagnostic assessment; language self-report as prior; start with medium-difficulty, adjust with Elo-like updates.
- **Evaluation**: Offline replay on historical data; online A/B tests with guardrails.

## Content Lifecycle
- **Human-authored**
  - Draft → Review → Approve → Publish; maintain versions and diff; support localized variants.
  - Linting and compliance checks at PR time; SBOMs for any binaries/media.
- **AI-generated**
  - Prompt templates with constraints; few-shot examples; deterministic or low-variance decoding.
  - Automated checks: toxicity, bias, reading level, originality. Watermark or provenance metadata.
  - Human-in-the-loop: queue for review; edits captured as new versions.

## Security Posture (OSS-first)
- **Identity & AuthN**: OIDC/OAuth2 via Keycloak; short-lived access tokens; refresh rotation; device- and geo-based risk scoring (later).
- **AuthZ**: RBAC for roles + ABAC for attributes; OPA sidecar and centralized policy repo; policy-as-code with CI checks.
- **Transport**: mTLS between services (SPIFFE/SPIRE identities), TLS 1.3 edge; HSTS, CSP, and hardened headers.
- **Data Protection**: At-rest encryption; field/column encryption for PII; per-tenant keys via KMS (HashiCorp Vault or AWS KMS); periodic rotation.
- **Secrets**: Vault/Kubernetes Secrets with sealed-secrets; no secrets in code or CI logs.
- **Input/Output**: JSON schema validation; output encoding to prevent XSS; SSRF/XXE disabled.
- **Rate limiting & DoS**: At API gateway and per-route; adaptive rules; circuit breakers and timeouts.
- **Audit & Forensics**: Immutable audit logs (WORM/S3 Object Lock); signed logs; admin actions require MFA and are always audited.
- **Supply Chain**: SLSA level ≥3 targets; SBOM (Syft), vuln scans (Trivy), signed images/artifacts (Cosign), dependency pinning, Renovate bot.
- **Compliance (future)**: GDPR/CCPA readiness, data subject rights workflows, data residency configuration.

## Observability & Reliability
- **Telemetry**: OpenTelemetry for traces/metrics/logs; exemplar dashboards (Grafana) and alerts (Alertmanager/PagerDuty-compatible).
- **SLIs/SLOs**: Availability and latency per service; recommendation freshness; index lag; error budgets.
- **Readiness**: Health endpoints, graceful shutdown, backpressure; chaos/probe testing in non-prod.

## Deployment & Environments
- **Packaging**: Docker for all services; multi-arch builds; SBOM and image signing.
- **Orchestration**: Kubernetes with a lightweight service mesh (Linkerd or Istio) for mTLS and traffic policy.
- **Envs**: dev (MinIO/Meilisearch/NATS), staging, prod. IaC via Terraform; GitOps via ArgoCD/Flux.
- **CI/CD**: GitHub Actions; test, scan, build, sign, promote; policy checks gate deploys.

## Technology Choices (proposed)
- **Frontend**: Next.js (App Router), TypeScript, Tailwind + shadcn/ui, React Query/Server Actions.
- **BFF/API**: Start with Next.js API routes; evolve to NestJS (TypeScript) as a separate gateway if needed.
- **Core Services**: TypeScript (NestJS) or Go for high-throughput; Python for ML in Recommendation/Assessment.
- **DB**: PostgreSQL, Prisma/Drizzle; Redis for cache/queues; Meilisearch for content search.
- **Eventing**: NATS (MVP), Kafka later if needed.
- **Object Storage**: S3/MinIO.
- **LLM/AI**: Provider-agnostic (OpenAI/Azure/Anthropic); local via Ollama for dev; guardrails (GuardrailsAI/Pydantic/JSON schema).

## API Snapshot (illustrative)
- `GET /api/me` → Profile
- `GET /api/content?skill=...&level=...&q=...` → Browse
- `POST /api/attempts` → Submit response (scoring async via event)
- `GET /api/recommendation/next` → Next best activity
- `GET /api/progress/:skillId` → Mastery and history
- `POST /api/content` (author) → Create draft; `POST /api/content/:id/publish` (reviewer)

## Roadmap (phased)
- **Phase 0 (Week 0-2)**: Monorepo layout, Keycloak integration, Postgres, minimal schemas, telemetry, CI with scans and SBOM.
- **Phase 1 (Week 2-6)**: MVP learning loop: browse → attempt → progress → basic rules-based recommendation; human-authored content only.
- **Phase 2 (Week 6-10)**: AI content pipeline with review and safety; assessment improvements; search and free navigation.
- **Phase 3 (Week 10-16)**: Probabilistic mastery model + contextual bandits; experimentation; notifications.
- **Phase 4+**: Service extraction (if warranted), multi-tenant isolation, regionalization, mobile and offline.

## Open Questions (please confirm)
1. Target audience and scope? K‑12, higher-ed, adult learners, exam prep?
2. Initial languages supported (English-only to start, or multi-language day one)?
3. Hosting preference: cloud-managed (AWS/GCP) vs fully self-hosted OSS stack?
4. Compliance needs now vs later (GDPR, COPPA, FERPA)? Any data residency constraints?
5. Single-tenant vs multi-tenant from the start?
6. Content licensing model and contribution workflow for external authors?
7. Moderation policy for AI-generated content and learner submissions?
8. Offline/mobile-first requirements?
9. Role matrix: learner, author, reviewer, admin—any others (parent/teacher)?
10. Budget for managed services vs OSS components?


