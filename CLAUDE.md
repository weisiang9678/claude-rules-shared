# Project Context

Python monorepo using Polylith architecture. See Core Rules for concepts and conventions.

## Rules & Skills

- **Rules** (`.claude/rules/00-core.md`) — Always-loaded conventions
- **Skills** (`.claude/skills/`) — On-demand procedures, auto-discovered via YAML frontmatter

### Workflow Chain

```
/design
  → approved plan saved to plans/YYYY-MM-DD-<topic>.md
    → /implement
      → phase-scoped verification (verification-before-completion) at each boundary
```

The [00-core.md](.claude/rules/00-core.md) file contains essential conventions that apply to all tasks:
- Polylith architecture concepts
- Tool versions (Python 3.13, uv 0.7.8)
- Python code standards (naming, imports, logging)
- Dependency subset rule (CRITICAL)
- Infrastructure/application separation
- Conventional commits

### Skills (Load On-Demand)

Skills provide detailed procedures and examples for specific tasks. They load automatically when relevant.

| Skill | Trigger | What It Provides |
|-------|---------|------------------|
| `polylith-new-brick` | Creating components, bases, or projects | Directory structure, pyproject.toml templates |
| `dependency-management` | Adding/updating dependencies, pyproject.toml issues | Workspace vs project patterns, troubleshooting |
| `deploy-cloud-functions` | Deploying to Cloud Functions | copy.sh scripts, requirements.txt generation |
| `deploy-cloudrun-service` | Deploying Cloud Run Services | Dockerfile patterns, Kustomize service manifests |
| `deploy-cloudrun-job` | Deploying Cloud Run Jobs | Job YAML, Kustomize overlays, scheduler setup |
| `deploy-dataflow-job` | Deploying Dataflow Jobs | Flex Templates, blue-green deployment, metadata.json |
| `github-actions-cicd` | Creating/modifying workflows | Workflow YAML examples, uv patterns |
| `pulumi-infrastructure` | Infrastructure provisioning | Pulumi patterns, two-stack architecture |
| `semantic-release` | Release workflows, versioning | Configuration, triggering releases |
| `pre-commit-hooks` | Setting up git hooks | .pre-commit-config.yaml template |

---

## Glossary

### Polylith Architecture

| Term | Definition |
|------|------------|
| **Workspace** | Top-level monorepo with `pyproject.toml`, `uv.lock`, `workspace.toml` |
| **Component** | Reusable brick in `components/{namespace}/` — stateless, single responsibility |
| **Base** | Entry point brick in `bases/{namespace}/` — composes components |
| **Project** | Deployable unit in `projects/` — specifies which bricks to include |
| **Namespace** | Python package grouping related bricks (e.g., `pipeline`, `asset`) |

### Infrastructure & Deployment

| Term | Definition |
|------|------------|
| **2-Stack Pattern** | Infrastructure (Pulumi) separate from application (Docker/Kustomize) |
| **Stack 1** | Foundational resources: service accounts, buckets, IAM, Artifact Registry |
| **Stack 2** | Application deployment: Docker images, Cloud Run manifests |
| **Kustomize** | Configuration management for environment-specific Cloud Run manifests |
| **Flex Template** | Containerized Dataflow job definition with metadata for parameterized deployment |
| **Blue-Green Deployment** | Deploy new job (green), wait for healthy, drain old jobs (blue) |

### Tools

| Tool | Purpose |
|------|---------|
| **uv** (0.7.8) | Fast Python package manager replacing pip/poetry |
| **Pulumi** | Infrastructure as Code for GCP resources |
| **Polylith CLI** | Create and manage bricks (`poly create component`) |
