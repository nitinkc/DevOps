# Agent Guide — DevOps Learning Workspace

## Purpose

Umbrella repo of learning submodules. Optimize changes for the long-term goal:
a complete, hands-on DevOps curriculum from Docker to production delivery on
GCP/GKE — theory + runnable labs + deliberate-break drills, not just notes.

## Repo layout

| Path | Type | Role |
|------|------|------|
| `DockerConcepts/` | submodule | Docker fundamentals → compose |
| `DevOpsLearning/` | submodule | Theory spine + labs, interview prep |
| `KubernetesConcepts/` | submodule | Flagship MkDocs lab site (Docker→Minikube→GKE) |
| `HelmCharts/` | submodule | Helm + Flux + interview bank |
| `terraform-concepts/` | submodule | TF basics → GCP → 5-root Acme multirepo |
| `gitlab-spring-app-flux/` | submodule | CI/CD + GitOps lab repo (GitLab pipeline track) |
| `jenkins-learning/` | submodule | Spring Boot app + Jenkinsfile track (secondary) |
| `go-k8s-server/`, `k8s-frontend/` | submodules | Toy apps reused by pipeline/deploy labs |

`README.md` contains the curriculum map and recommended study order.

## Submodule rules

- Every subdirectory is its own git repo (`.git` file, not dir). Run git
  commands from inside the submodule, or use `git -C <submodule>`.
- Commit per-submodule; then bump the pointer in this umbrella repo as a
  separate commit. Never commit submodule contents from the umbrella repo.
- Each repo may have its own `AGENTS.md` (KubernetesConcepts and
  terraform-concepts do) — the closest one wins.
- `.venv/` and `node_modules/` dirs exist inside submodules — exclude them
  from searches and never edit them.

## Pedagogy conventions (apply to every new lab/doc)

- Reuse the Architect's Loop: **Why** (design decision) → **Visual blueprint**
  → **Hands-on build & verify** (copy-pasteable, expected output) →
  **Deliberate break & triage drill**.
- Keep the local↔GCP twin mapping explicit (rosetta-stone style).
- Every lab ends with a verify section; docs end with assessment/quiz where
  the repo already has that convention.
- Register new pages in the repo's `mkdocs.yml` nav in the same commit.

## Safety

- `terraform-concepts/acme-sampleapp-multirepo` creates **billable GCP
  resources**. No `terraform apply`/`destroy` or `restore-session.sh
  --bootstrap` without explicit user approval. See that repo's AGENTS.md.
- Never commit credentials, state, plan files, generated secrets, or real
  organization identifiers.

## Verification

- Per-repo: `mkdocs build --strict`, `terraform fmt -check -recursive`,
  `bash -n` on scripts, `helm lint` on charts.
- YAML sanity: `kubectl apply --dry-run=client -f` (or `--dry-run=server` when
  a cluster is up) on new manifests.
- GitLab CI YAML: `glab ci lint` (or the /lint API) before committing a
  `.gitlab-ci.yml`.
