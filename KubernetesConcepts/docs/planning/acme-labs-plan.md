# Acme Labs — GKE + Terraform + GitLab + Flux Platform Lab

Task-wise plan, hints, commands and starter Terraform.
Companion deck: `acme-labs-design.html` (diagrams + high-level design).

> **Assumed already known (not re-taught):** Terraform core (state, `for_each`, import, GCS backend), Workload Identity concept, Consul pub/sub pattern, `kubectl`/Helm basics, GKE IAM/RBAC. This plan spends its time on **networking, GitLab-tiered Terraform, identity federation, sidecars/observability and GitOps ordering**.

---

## 0. Assumptions (change these first if wrong)

| # | Assumption | If wrong |
|---|---|---|
| A1 | You use **gitlab.com Free** (not self-managed) | Change the OIDC issuer in the WIF provider to your host |
| A2 | Your Google account may have **no Organization** (personal gmail) → no Folders, no org policies | `bootstrap` creates **projects only**; the "folder" level of the corporate diagram is simulated with naming. If you do have an org, add `google_folder` resources (see P1 note) |
| A3 | Existing project `my-devops-journey-502420` = **non-prod (np)**. A second, almost-empty project = **prod (p)** to prove permission boundaries | Skip prod; you lose the protected-branch lab (P2) |
| A4 | Region `us-central1`, zone `us-central1-a` (free-tier e2-micro region, cheapest) | Change `var.region/zone` |
| A5 | "Three separate VPN" = **three isolated network domains** (not literally 3 VPCs). See D1 | See D1 stretch S3/S6 |

---

## 1. Architecture decisions (with reasons — these are the "why" you should be able to defend)

### D1 — Can ONE cluster serve 3 isolated domains? **Yes.**
A GKE cluster lives in exactly **one VPC**, so you cannot give three domains three VPCs without three clusters. But "domain isolation" does not need VPCs. Use layered isolation:

| Layer | Mechanism | Stops what |
|---|---|---|
| Cloud IAM | One GSA per domain via Workload Identity | A pod using another domain's cloud permissions |
| L3/L4 | Kubernetes `NetworkPolicy` (GKE **Dataplane V2**) default-deny per namespace | Pod→pod traffic not explicitly allowed |
| L7 identity | Istio mTLS + `AuthorizationPolicy` on SPIFFE identity (SA) | A permitted network path used by the wrong caller |
| Capacity | `ResourceQuota` / `LimitRange` per namespace | Noisy neighbour |
| Network (stretch) | Per-domain node pool with its **own pod secondary range** + node SA/tags → VPC firewall by CIDR | Lets you write real VPC firewall rules per domain (S3) |

If you ever need *real* VPC separation: three clusters (or GKE multi-network / Shared VPC with 3 subnets). That is stretch **S6**.

### D2 — Cluster flavour
GKE **Standard, zonal, private nodes, Dataplane V2, Workload Identity, Gateway API**. Zonal keeps you on the free-tier management-fee credit (one zonal/Autopilot cluster per billing account). **Spot** `e2-standard-2` ×2–3 nodes. Standard (not Autopilot) because you want to see node tags, routes, NAT and pod ranges — Autopilot hides them.

### D3 — Where do "service accounts + cross-stack values" live? (your Consul/other question)
- **Identities are deterministic names**, not looked-up values: `pipeline-np@…`, `orders-app@…`. A name you can compute needs no lookup service.
- **Cross-stack outputs** (VPC self-link, subnet names): use **GCS remote state + `terraform_remote_state`**. Reason: zero extra infra, works with GitLab shared runners, IAM-protected.
- **Consul = optional stretch S4.** A private Consul can't be reached by gitlab.com shared runners; you'd need a self-hosted runner on the free-tier e2-micro in the VPC. Great learning (it forces you through the network doors), poor *day-one* choice.
- Secrets (later): **Secret Manager** (free tier 6 active versions) + External Secrets Operator. Don't run Vault for a lab.

### D4 — Mesh / sidecars
Istio **sidecar mode** (deliberately; ambient mode would remove the sidecars you want to study). Per application pod:

| Container | Role | Log path to Cloud Logging |
|---|---|---|
| `app` | Your service, JSON logs to stdout **and** a file | stdout → GKE node agent → `_Default` bucket |
| `istio-proxy` (Envoy) | mTLS, authz, L7 metrics | JSON access log to stdout → same path |
| `otel-collector` | OTLP receiver → `googlecloud` exporter (logs, traces, metrics) | Direct API call, via Workload Identity |
| `fluent-bit` | Tails the app's log **file** on a shared `emptyDir` | `stackdriver` output, direct API call, via Workload Identity |

Three patterns for the same goal (stdout agent / file-tail sidecar / OTLP sidecar) — you'll see why enterprises use each.
Future sidecar: Cloud SQL Auth Proxy (S1).

### D5 — Central logging
One **log bucket `central-app-logs`** (Log Analytics enabled) fed by a **Log Router sink** filtered on the three domain namespaces; an exclusion on `_Default` avoids storing logs twice. Cloud Logging free allotment ≈ 50 GiB/project/month (verify current figure). Domain separation inside the central bucket is by label/namespace, not by bucket.

### D6 — GitLab tiering and where Terraform modules live
Mirrors your corporate layout. Modules in Tier 1 (`platform/terraform-modules`), consumed by git tag (`?ref=v0.1.0`) so Tier 2/3 never copy code. CI templates in Tier 1 (`platform/ci-templates`). Group-level CI variables inherit downward.

### D7 — Auth from GitLab to GCP: **Workload Identity Federation, no JSON keys** (same "no SA keys" guardrail as the corporate screenshot).

### D8 — Flux owns Kubernetes objects; Terraform owns GCP objects
Terraform never `kubectl apply`s. CI never needs access to the Kubernetes API → the GKE control-plane endpoint can be locked to your own IP. The only handoff between the two is **names** (`gke-nodes`, `orders-app`, ns `dom-orders`).

---

## 2. Naming + IPAM (decide *before* writing any Terraform)

| Thing | Value |
|---|---|
| np project | `my-devops-journey-502420` |
| prod project | `acme-labs-p-<suffix>` (new) |
| GitLab root group | `acme-labs-<handle>` (must be globally unique — write it as `$ROOT` below) |
| VPC | `acme-main` (custom mode) |
| Cluster | `acme-np` (zonal, `us-central1-a`) |
| Namespaces | `dom-orders`, `dom-payments`, `dom-inventory` |
| KSA → GSA | `orders`→`orders-app`, `payments`→`payments-app`, `inventory`→`inventory-app` |
| Node tags | `gke-node`, `allowed-open-nat`, `iap-ssh` |

| Range | CIDR | Purpose | Corporate analogue |
|---|---|---|---|
| `gke-main` nodes | `10.10.0.0/24` | Node primary IPs | Enterprise 10.x |
| `gke-pods` (secondary) | `10.20.0.0/20` | Pod IPs (with `max_pods_per_node=64` → /25 per node → 32 nodes max) | GKE pod CIDR ≥ /22 |
| `gke-svcs` (secondary) | `10.30.0.0/24` | ClusterIPs | GKE svc CIDR |
| `ops` | `10.10.1.0/28` | Bastion/test VM | — |
| GKE master | `10.10.250.0/28` | Private control plane (/28 mandatory) | 10.3.82.0/28 |
| `proxy-only-subnet` | `10.10.254.0/23` | Envoy pool for internal L7 LBs (`REGIONAL_MANAGED_PROXY`) | 5.1.0.0/23 |
| PSA (stretch) | `100.64.0.0/20` | Cloud SQL private services access | 100.64.0.0/16 |
| `onprem-sim` (stretch) | `192.168.100.0/24` | Second VPC faking an on-prem site | den3 / NCC spokes |

Rules to internalize: ranges must not overlap with **anything you may ever connect** (peering, VPN, NCC); pod/service secondary ranges are painful to change after cluster creation; PSA range ≠ anything routable on-prem.

---

## 3. GitLab layout (screenshot 2, renamed)

```
acme-labs-<handle>/                       ← root group  (≙ corporate-applications)
├─ platform/                              ← TIER 1  "city power grid & roads"
│  ├─ bootstrap/                          projects, APIs, state buckets, WIF pool, pipeline SAs
│  ├─ network/                            VPC, subnets, routes, NAT, firewall, DNS, PSA
│  ├─ gke-platform/                       cluster, node pools, node SA, Artifact Registry
│  ├─ observability/                      log bucket, sinks, log-based metrics
│  ├─ terraform-modules/                  versioned modules (git tags)
│  ├─ ci-templates/                       shared .gitlab-ci includes
│  └─ contract-bus/        (optional S4)  Consul on e2-micro
├─ domain-shared/                         ← TIER 2  "shared utilities & blueprints"
│  ├─ shared-services/                    (later) Cloud SQL, Pub/Sub, Memorystore
│  └─ configurations/
│     ├─ helm-charts/                     acme-microservice chart (OCI-published)
│     └─ flux-releases/                   GitOps repo Flux watches
└─ applications/                          ← TIER 3  "storefronts"
   ├─ orders/     { orders-svc, orders-infra }
   ├─ payments/   { payments-svc, payments-infra }
   └─ inventory/  { inventory-svc, inventory-infra }
```

**Group-level CI/CD variables (set once on the root, inherited everywhere):**

| Variable | Scope | Protected? |
|---|---|---|
| `WIF_PROVIDER` (`projects/<NUM>/locations/global/workloadIdentityPools/gitlab-pool/providers/gitlab-oidc`) | root | no |
| `GCP_PROJECT_NP`, `GCP_REGION`, `TF_STATE_BUCKET_NP` | root | no |
| `SA_NP`, `SA_P_RO` | root | no |
| `SA_P`, `TF_STATE_BUCKET_P` | root | **yes** (only protected branches see them) |
| `AR_REPO` (Artifact Registry path), `AR_PUSH_SA` | **`applications/` only** | no |

> **Hands-on proof (mirrors the screenshot's "outside applications/ → 404"):** create a repo directly under the root, run an image build job: it fails because `AR_REPO` is not inherited there. Folder path is a permission boundary, not decoration.

Create the groups:

```bash
export GL_PAT=...            # PAT with api scope
ROOT=acme-labs-yourhandle
api() { curl -s -H "PRIVATE-TOKEN: $GL_PAT" "$@"; }
rid=$(api -X POST https://gitlab.com/api/v4/groups -d "name=$ROOT&path=$ROOT&visibility=private" | jq .id)
for g in platform domain-shared applications; do
  api -X POST https://gitlab.com/api/v4/groups -d "name=$g&path=$g&parent_id=$rid&visibility=private" | jq '{id,full_path}'
done
# then: configurations under domain-shared; orders/payments/inventory under applications
```

---

## 4. Order of implementation — and the logic behind it

```
P0 tooling/groups
 └ P1 bootstrap (local)        ← creates identity + state; nothing else can run in CI without it
    └ P2 CI + WIF              ← from now on every change is a merge request
       └ P3 network v1         ← IPAM must exist BEFORE the cluster (pod ranges are baked in)
          └ P4 GKE + registry  ← needs subnet/secondary ranges + node SA
             └ P5 Flux         ← needs a cluster; installs controllers in dependency order
                ├ P6 observability  ← sink + sidecar config; verify before apps exist
                └ P7 3 domains      ← apps need mesh (P5) + log path (P6) + identities
                   └ P8 close the doors (delete default route)  ← only after you KNOW every flow
                      └ P9 internal Gateway (proxy-only subnet)
                         └ P10 stretch menu (PSA/Cloud SQL, hybrid VPN, node-pool-per-domain, Consul …)
```
**Why close the default route so late (P8)?** A deleted default route breaks node bootstrap, Flux→GitLab, chart downloads and Docker Hub pulls. You first build everything *with training wheels*, record every flow, then remove the wheels and repair each break deliberately. That is where the hidden nuances surface.

Effort guide at several hrs/week: P0–P2 ≈ 1 week; P3–P4 ≈ 1 week; P5–P7 ≈ 1.5 weeks; P8–P9 ≈ 1 week; stretch as desired.

---

## P0 — Tooling, billing guard-rails, groups

**Goal:** safe sandbox; spending alarm; groups exist.

```bash
gcloud auth login && gcloud auth application-default login
gcloud config set project my-devops-journey-502420
gcloud version; terraform version; kubectl version --client; helm version; flux --version
# budget alarm (billing admin needed)
gcloud billing budgets create --billing-account=<ID> --display-name=acme-labs \
  --budget-amount=20USD --threshold-rule=percent=0.5 --threshold-rule=percent=1.0
```
Hints: install `istioctl` (for `istioctl analyze`, `istioctl proxy-status`) and `jq`. Create groups with §3 script.
**Verify:** `curl -s -H "PRIVATE-TOKEN: $GL_PAT" https://gitlab.com/api/v4/groups/$ROOT | jq .full_path`

---

## P1 — Tier-1 `bootstrap` (run **locally** first; it creates the thing CI will use)

**Goal:** APIs, state buckets, prod project, WIF pool, pipeline service accounts with *different trust levels*.
**Nuance revealed:** federation trust is expressed as **attribute mappings + principalSets**; "protected-branch-only" is a CEL string, not a GitLab feature you toggle in GCP.

```hcl
# platform/bootstrap/main.tf
terraform {
  required_version = ">= 1.9"
  required_providers { google = { source = "hashicorp/google", version = ">= 6.0, < 8.0" } }
}
variable "project_id"        { type = string }                  # np, existing
variable "prod_project_id"   { type = string }                  # new
variable "billing_account"   { type = string }
variable "gitlab_root_group" { type = string }                  # e.g. acme-labs-yourhandle
variable "region"            { default = "us-central1" }
provider "google" { project = var.project_id  region = var.region }

locals {
  apis = [
    "compute.googleapis.com", "container.googleapis.com", "iam.googleapis.com",
    "iamcredentials.googleapis.com", "sts.googleapis.com", "cloudresourcemanager.googleapis.com",
    "logging.googleapis.com", "monitoring.googleapis.com", "cloudtrace.googleapis.com",
    "artifactregistry.googleapis.com", "dns.googleapis.com", "secretmanager.googleapis.com",
    "servicenetworking.googleapis.com", "networkmanagement.googleapis.com",
  ]
}
resource "google_project_service" "np" {
  for_each = toset(local.apis)
  service  = each.value
  disable_on_destroy = false
}

# prod: no org => no parent block. With an org, add org_id/folder_id here.
resource "google_project" "prod" {
  name = "acme-labs-p"  project_id = var.prod_project_id
  billing_account = var.billing_account
  deletion_policy = "DELETE"
}
resource "google_project_service" "prod" {
  for_each = toset(["iam.googleapis.com", "storage.googleapis.com", "cloudresourcemanager.googleapis.com"])
  project = google_project.prod.project_id  service = each.value
  disable_on_destroy = false
}

# --- state buckets (the org-policy-friendly settings you already know) ---
resource "google_storage_bucket" "state_np" {
  name = "${var.project_id}-tfstate-np"  location = var.region
  uniform_bucket_level_access = true  public_access_prevention = "enforced"
  versioning { enabled = true }
  lifecycle_rule {
    condition { num_newer_versions = 10 }
    action    { type = "Delete" }
  }
}
resource "google_storage_bucket" "state_p" {
  project = google_project.prod.project_id
  name = "${var.prod_project_id}-tfstate-p"  location = var.region
  uniform_bucket_level_access = true  public_access_prevention = "enforced"
  versioning { enabled = true }
  depends_on = [google_project_service.prod]
}

# --- WIF: GitLab OIDC -> GCP ---
resource "google_iam_workload_identity_pool" "gitlab" {
  workload_identity_pool_id = "gitlab-pool"
  depends_on = [google_project_service.np]
}
resource "google_iam_workload_identity_pool_provider" "gitlab" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.gitlab.workload_identity_pool_id
  workload_identity_pool_provider_id = "gitlab-oidc"
  attribute_mapping = {
    "google.subject"            = "assertion.sub"
    "attribute.project_path"    = "assertion.project_path"
    "attribute.root_group"      = "assertion.project_path.split('/')[0]"
    # composite attribute: only equals "<root>:main:true" for a PROTECTED main branch
    "attribute.root_group_ref"  = "assertion.project_path.split('/')[0] + ':' + assertion.ref + ':' + assertion.ref_protected"
  }
  attribute_condition = "assertion.project_path.startsWith('${var.gitlab_root_group}/')"
  oidc { issuer_uri = "https://gitlab.com" }   # self-managed: your https://gitlab.host
}

# --- pipeline service accounts: three trust levels ---
locals {
  pipeline_sas = {
    "pipeline-np"   = var.project_id
    "pipeline-p"    = google_project.prod.project_id
    "pipeline-p-ro" = google_project.prod.project_id
  }
  pool = google_iam_workload_identity_pool.gitlab.name
  principals = {
    "pipeline-np"   = "principalSet://iam.googleapis.com/${local.pool}/attribute.root_group/${var.gitlab_root_group}"
    "pipeline-p-ro" = "principalSet://iam.googleapis.com/${local.pool}/attribute.root_group/${var.gitlab_root_group}"
    "pipeline-p"    = "principalSet://iam.googleapis.com/${local.pool}/attribute.root_group_ref/${var.gitlab_root_group}:main:true"
  }
  sa_roles = {   # lab-grade; tightening these is a later exercise
    "pipeline-np"   = ["roles/editor", "roles/resourcemanager.projectIamAdmin", "roles/iam.serviceAccountAdmin", "roles/container.admin"]
    "pipeline-p"    = ["roles/editor", "roles/resourcemanager.projectIamAdmin", "roles/iam.serviceAccountAdmin"]
    "pipeline-p-ro" = ["roles/viewer"]
  }
  sa_role_pairs = merge([for sa, roles in local.sa_roles : { for r in roles : "${sa}|${r}" => { sa = sa, role = r } }]...)
}
resource "google_service_account" "pipeline" {
  for_each = local.pipeline_sas
  project = each.value  account_id = each.key
}
resource "google_service_account_iam_member" "wif" {
  for_each = local.principals
  service_account_id = google_service_account.pipeline[each.key].name
  role = "roles/iam.workloadIdentityUser"  member = each.value
}
resource "google_project_iam_member" "pipeline" {
  for_each = local.sa_role_pairs
  project = local.pipeline_sas[each.value.sa]
  role = each.value.role
  member = "serviceAccount:${google_service_account.pipeline[each.value.sa].email}"
}

output "wif_provider" { value = google_iam_workload_identity_pool_provider.gitlab.name }
output "pipeline_sa"  { value = { for k, v in google_service_account.pipeline : k => v.email } }
output "state_buckets" { value = { np = google_storage_bucket.state_np.name, p = google_storage_bucket.state_p.name } }
```
```bash
cd platform/bootstrap && terraform init && terraform apply \
  -var project_id=my-devops-journey-502420 -var prod_project_id=acme-labs-p-$RANDOM \
  -var billing_account=XXXXXX-XXXXXX-XXXXXX -var gitlab_root_group=$ROOT
# move bootstrap's own state into the bucket it just made (you know this dance):
#   add backend "gcs" {} then: terraform init -migrate-state -backend-config="bucket=<np bucket>" -backend-config="prefix=platform/bootstrap"
# publish outputs as group variables (see §3). Example:
GID=<root group id>
api -X POST https://gitlab.com/api/v4/groups/$GID/variables \
  -F key=WIF_PROVIDER -F value="$(terraform output -raw wif_provider)"
api -X POST https://gitlab.com/api/v4/groups/$GID/variables \
  -F key=SA_P -F value="pipeline-p@<prod>.iam.gserviceaccount.com" -F protected=true
```
Hints: if `google_project` fails on billing, verify `billing_account` and your `roles/billing.user`. Org policy 412? That's a constraint, not a Terraform bug.
**Verify:** `gcloud iam workload-identity-pools providers describe gitlab-oidc --workload-identity-pool=gitlab-pool --location=global --format='yaml(attributeCondition,attributeMapping)'`
**Gate G1:** you can explain, without notes, why `pipeline-p`'s principalSet cannot be matched by a feature-branch token.

---

## P2 — Tier-1 CI templates + first pipeline through WIF

**Goal:** `platform/ci-templates` provides one reusable Terraform pipeline; `platform/terraform-modules` is readable by jobs.

```yaml
# platform/ci-templates/terraform.yml
.tf:
  image: { name: hashicorp/terraform:1.9, entrypoint: [""] }
  variables:
    TF_IN_AUTOMATION: "true"
    TF_ROOT: "."
    TF_SA: "$SA_NP"
    TF_STATE_BUCKET: "$TF_STATE_BUCKET_NP"
  id_tokens:
    GCP_OIDC:
      aud: "https://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/gitlab-pool/providers/gitlab-oidc"
      # if your GitLab version expands variables in aud you can use ${WIF_PROVIDER}; otherwise hardcode as above
  before_script:
    - echo "$GCP_OIDC" > /tmp/oidc.jwt
    - |
      cat > /tmp/wif.json <<EOF
      {"type":"external_account","audience":"//iam.googleapis.com/${WIF_PROVIDER}",
       "subject_token_type":"urn:ietf:params:oauth:token-type:jwt",
       "token_url":"https://sts.googleapis.com/v1/token",
       "service_account_impersonation_url":"https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/${TF_SA}:generateAccessToken",
       "credential_source":{"file":"/tmp/oidc.jwt"}}
      EOF
    - export GOOGLE_APPLICATION_CREDENTIALS=/tmp/wif.json
    # private module repos in Tier 1: the job token needs an allow-list entry in terraform-modules
    - git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/".insteadOf "https://gitlab.com/"
    - cd "$TF_ROOT"
    - terraform init -input=false -backend-config="bucket=${TF_STATE_BUCKET}" -backend-config="prefix=${TF_STATE_PREFIX:-$CI_PROJECT_PATH_SLUG}"

stages: [validate, plan, apply]
tf:validate:
  extends: .tf
  stage: validate
  script: [terraform fmt -check -recursive, terraform validate]
tf:plan:
  extends: .tf
  stage: plan
  script: [terraform plan -input=false -out=plan.tfplan]
  artifacts: { paths: ["$TF_ROOT/plan.tfplan"], expire_in: 1 week }
tf:apply:
  extends: .tf
  stage: apply
  needs: [tf:plan]
  rules: [{ if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH', when: manual }]
  script: [terraform apply -input=false plan.tfplan]
```
Consumer repo `.gitlab-ci.yml`:
```yaml
include:
  - project: acme-labs-yourhandle/platform/ci-templates
    file: terraform.yml
```
Hints: in `platform/terraform-modules` → Settings → CI/CD → *Job token permissions* → add the groups that consume it. Tag modules (`git tag v0.1.0`) and pin `?ref=`. Shared runners have a monthly minute quota (≈400 on Free; verify) — keep jobs lean, cache nothing fancy.
**Break-it exercises**
1. Run a job on a **feature branch** with `TF_SA=$SA_P`: STS exchange must be **denied**. Read the error and map it to the `root_group_ref` attribute.
2. Switch `TF_SA` to `$SA_P_RO`: succeeds with read-only (the "feature branches get read-only plan view" behaviour).
3. Protect `main` ⇒ `ref_protected=true` ⇒ `SA_P` works from `main` only.
**Gate G2:** all three behave as predicted *before* you run them.

---

## P3 — Tier-1 `network` v1 (the main learning stack, with training wheels)

**Goal:** custom VPC with IPAM from §2, PGA, Cloud NAT with 2 static IPs, tag-scoped routes, firewall, private DNS for Google APIs. A variable `training_wheels` keeps a normal default route until P8.
**Nuances revealed (write each down in your own words):**
- Cloud NAT only translates traffic whose route's next hop is `default-internet-gateway`.
- `delete_default_routes_on_create` is a **create-time** flag — toggling later forces VPC replacement; so manage your own default route as a resource instead.
- Network tags / SAs are **node-level**: pods ride on the node's identity.
- GKE pod range ≠ node range: NAT must be told `ALL_IP_RANGES` or pods have no egress.

```hcl
# platform/network/main.tf  (backend "gcs" { prefix = "platform/network" })
variable "project_id" {}
variable "region" { default = "us-central1" }
variable "training_wheels" { default = true }                # flip to false in P8
variable "vip_allowlist" {                                   # Door 2: exact /32s
  type = map(string)
  default = {}   # e.g. { gitlab-example = "203.0.113.10/32" }
}

resource "google_compute_network" "main" {
  name = "acme-main"
  auto_create_subnetworks         = false
  routing_mode                    = "GLOBAL"
  delete_default_routes_on_create = true      # GCP's implicit 0.0.0.0/0 is removed at birth
}

# ---------- subnets ----------
resource "google_compute_subnetwork" "gke_main" {
  name = "gke-main"  region = var.region  network = google_compute_network.main.id
  ip_cidr_range = "10.10.0.0/24"
  private_ip_google_access = true                           # Door 1 prerequisite
  secondary_ip_range { range_name = "gke-pods"  ip_cidr_range = "10.20.0.0/20" }
  secondary_ip_range { range_name = "gke-svcs"  ip_cidr_range = "10.30.0.0/24" }
  log_config { aggregation_interval = "INTERVAL_5_MIN" flow_sampling = 0.1 metadata = "INCLUDE_ALL_METADATA" }
}
resource "google_compute_subnetwork" "ops" {
  name = "ops" region = var.region network = google_compute_network.main.id
  ip_cidr_range = "10.10.1.0/28"  private_ip_google_access = true
}
resource "google_compute_subnetwork" "proxy_only" {
  name = "proxy-only-subnet" region = var.region network = google_compute_network.main.id
  ip_cidr_range = "10.10.254.0/23"
  purpose = "REGIONAL_MANAGED_PROXY"  role = "ACTIVE"
}

# ---------- routes: the four doors live here ----------
locals { igw = "default-internet-gateway" }

resource "google_compute_route" "training_wheels" {          # removed in P8
  count = var.training_wheels ? 1 : 0
  name = "tw-default-internet" network = google_compute_network.main.name
  dest_range = "0.0.0.0/0" next_hop_gateway = local.igw priority = 1000
}
resource "google_compute_route" "allowed_open_nat" {         # Door 3: tag-gated default route
  name = "allowed-open-nat" network = google_compute_network.main.name
  dest_range = "0.0.0.0/0" next_hop_gateway = local.igw priority = 999
  tags = ["allowed-open-nat"]
}
resource "google_compute_route" "public_services" {          # Door 2: explicit /32s
  for_each = var.vip_allowlist
  name = "inet-${each.key}" network = google_compute_network.main.name
  dest_range = each.value next_hop_gateway = local.igw priority = 900
}
resource "google_compute_route" "private_google_vip" {       # Door 1 with a deleted default route
  name = "private-googleapis-vip" network = google_compute_network.main.name
  dest_range = "199.36.153.8/30" next_hop_gateway = local.igw priority = 900
}

# ---------- Cloud NAT: 2 static IPs so partners can allow-list you ----------
resource "google_compute_router" "main" {
  name = "acme-router" region = var.region network = google_compute_network.main.id
}
resource "google_compute_address" "nat" {
  count = 2  name = "nat-external-${count.index}" region = var.region
}
resource "google_compute_router_nat" "main" {
  name = "acme-nat" router = google_compute_router.main.name region = var.region
  nat_ip_allocate_option = "MANUAL_ONLY"
  nat_ips = google_compute_address.nat[*].self_link
  source_subnetwork_ip_ranges_to_nat = "LIST_OF_SUBNETWORKS"
  subnetwork {
    name = google_compute_subnetwork.gke_main.id
    source_ip_ranges_to_nat = ["ALL_IP_RANGES"]            # nodes AND pods
  }
  log_config { enable = true filter = "ERRORS_ONLY" }
}

# ---------- DNS: make *.googleapis.com / gcr.io / pkg.dev resolve to the private VIP ----------
locals { g_zones = { googleapis = "googleapis.com.", gcr = "gcr.io.", pkgdev = "pkg.dev." } }
resource "google_dns_managed_zone" "g" {
  for_each = local.g_zones
  name = "${each.key}-private" dns_name = each.value visibility = "private"
  private_visibility_config { networks { network_url = google_compute_network.main.id } }
}
resource "google_dns_record_set" "vip_a" {                   # private.googleapis.com
  managed_zone = google_dns_managed_zone.g["googleapis"].name
  name = "private.googleapis.com." type = "A" ttl = 300
  rrdatas = ["199.36.153.8", "199.36.153.9", "199.36.153.10", "199.36.153.11"]
}
resource "google_dns_record_set" "wild" {
  for_each = local.g_zones
  managed_zone = google_dns_managed_zone.g[each.key].name
  name = "*.${each.value}" type = "CNAME" ttl = 300
  rrdatas = ["private.googleapis.com."]
}

# ---------- firewall (VPC is deny-ingress by default; add only what you can justify) ----------
resource "google_compute_firewall" "iap_ssh" {
  name = "allow-iap-ssh" network = google_compute_network.main.name
  source_ranges = ["35.235.240.0/20"] target_tags = ["iap-ssh"]
  allow { protocol = "tcp" ports = ["22"] }
}
resource "google_compute_firewall" "lb_health" {
  name = "allow-lb-health" network = google_compute_network.main.name
  source_ranges = ["130.211.0.0/22", "35.191.0.0/16"] target_tags = ["gke-node"]
  allow { protocol = "tcp" }
}
resource "google_compute_firewall" "proxy_to_pods" {          # needed for internal L7 LB (P9)
  name = "allow-proxy-only-to-nodes" network = google_compute_network.main.name
  source_ranges = [google_compute_subnetwork.proxy_only.ip_cidr_range] target_tags = ["gke-node"]
  allow { protocol = "tcp" ports = ["8080", "8443", "15021"] }
}
resource "google_compute_firewall" "master_webhooks" {        # classic private-cluster trap
  name = "allow-gke-master-webhooks" network = google_compute_network.main.name
  source_ranges = ["10.10.250.0/28"] target_tags = ["gke-node"]
  allow { protocol = "tcp" ports = ["15017", "8443", "9443", "10250"] }  # istiod webhook, etc.
}

output "network_self_link" { value = google_compute_network.main.self_link }
output "subnet_name"       { value = google_compute_subnetwork.gke_main.name }
output "network_name"      { value = google_compute_network.main.name }
```
Hints:
- Use a throwaway **IAP-only VM** in `ops` (no external IP, tag `iap-ssh`) to test doors:
  ```bash
  gcloud compute instances create probe --zone us-central1-a --machine-type e2-micro \
    --subnet ops --no-address --tags iap-ssh,allowed-open-nat --image-family debian-12 --image-project debian-cloud
  gcloud compute ssh probe --zone us-central1-a --tunnel-through-iap -- curl -sI https://example.com
  ```
  Remove the `allowed-open-nat` tag and re-test: with `training_wheels=true` it still works (tw route); after P8 it won't. That contrast **is** the lesson.
- Network Intelligence → *Connectivity Tests* (`gcloud network-management connectivity-tests create …`) will tell you which route/firewall decides a packet.
**Verify:**
```bash
gcloud compute routes list --filter="network:acme-main" \
  --format="table(name,destRange,priority,nextHopGateway.basename(),tags.list())"
gcloud compute routers get-nat-mapping-info acme-router --region us-central1
```
**Gate G3:** predict the result of curling (a) `storage.googleapis.com`, (b) `example.com` from a tagged and an untagged VM, in both `training_wheels` states — then confirm.

---

## P4 — Tier-1 `gke-platform` + Artifact Registry

**Goal:** private zonal Standard cluster on the P3 ranges; node SA with least privilege; app registry + Docker Hub **mirror** (so image pulls use Door 1 instead of needing the internet).

```hcl
# platform/gke-platform/main.tf
data "terraform_remote_state" "net" {
  backend = "gcs"
  config  = { bucket = var.state_bucket, prefix = "platform/network" }
}
variable "project_id" {}
variable "state_bucket" {}
variable "region" { default = "us-central1" }
variable "zone"   { default = "us-central1-a" }
variable "admin_cidr" {}                       # YOUR public IP /32 — control plane allow-list

resource "google_service_account" "nodes" { account_id = "gke-nodes" }
resource "google_project_iam_member" "nodes" {
  for_each = toset(["roles/logging.logWriter", "roles/monitoring.metricWriter", "roles/monitoring.viewer",
                    "roles/stackdriver.resourceMetadata.writer", "roles/artifactregistry.reader"])
  project = var.project_id  role = each.value
  member = "serviceAccount:${google_service_account.nodes.email}"
}

resource "google_container_cluster" "np" {
  name = "acme-np"  location = var.zone  deletion_protection = false
  network    = data.terraform_remote_state.net.outputs.network_self_link
  subnetwork = data.terraform_remote_state.net.outputs.subnet_name
  remove_default_node_pool = true  initial_node_count = 1
  networking_mode = "VPC_NATIVE"
  datapath_provider = "ADVANCED_DATAPATH"                  # Dataplane V2 (Cilium): NetworkPolicy enforced
  ip_allocation_policy {
    cluster_secondary_range_name  = "gke-pods"
    services_secondary_range_name = "gke-svcs"
  }
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false                         # reachable from admin_cidr only (below)
    master_ipv4_cidr_block  = "10.10.250.0/28"
  }
  master_authorized_networks_config {
    cidr_blocks { cidr_block = var.admin_cidr  display_name = "admin" }
  }
  workload_identity_config { workload_pool = "${var.project_id}.svc.id.goog" }
  gateway_api_config { channel = "CHANNEL_STANDARD" }
  release_channel { channel = "REGULAR" }
  logging_config    { enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"] }
  monitoring_config { enable_components = ["SYSTEM_COMPONENTS"] }
}

resource "google_container_node_pool" "main" {
  name = "main"  cluster = google_container_cluster.np.name  location = var.zone
  autoscaling { min_node_count = 2 max_node_count = 3 }
  max_pods_per_node = 64
  node_config {
    machine_type = "e2-standard-2"  spot = true  disk_size_gb = 30  disk_type = "pd-standard"
    service_account = google_service_account.nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]
    tags = ["gke-node", "allowed-open-nat"]                 # NODE-level tags (pods inherit!)
    workload_metadata_config { mode = "GKE_METADATA" }
    shielded_instance_config { enable_secure_boot = true }
  }
}

resource "google_artifact_registry_repository" "apps" {
  repository_id = "apps"  format = "DOCKER"  location = var.region
}
resource "google_artifact_registry_repository" "dockerhub" {
  repository_id = "dockerhub-mirror"  format = "DOCKER"  location = var.region
  mode = "REMOTE_REPOSITORY"
  remote_repository_config { docker_repository { public_repository = "DOCKER_HUB" } }
}
```
```bash
gcloud container clusters get-credentials acme-np --zone us-central1-a --project my-devops-journey-502420
kubectl get nodes -o wide                         # INTERNAL-IP in 10.10.0.0/24, no EXTERNAL-IP
kubectl get pods -A -o wide | head                # pod IPs in 10.20.0.0/20
```
Hints: spot nodes can vanish — fine for a lab. If nodes don't become Ready, it's almost always egress (P3) — check NAT logs / route list.
**Nuances revealed:** why `enable_private_endpoint=false` + authorized networks (so *you* can `kubectl`, nobody else can); why nodes have no external IP yet can still pull images (NAT or PGA?).
**Gate G4:** you can say which door each of these uses: `gke.gcr.io` system image, `docker.io/istio/pilot`, `registry.k8s.io`, the AR mirror.

---

## P5 — Tier-2 `flux-releases` + Flux bootstrap (the order matters)

**Goal:** Flux installed; infra layered via `dependsOn`; nothing applied by hand after bootstrap.

```bash
export GITLAB_TOKEN=...   # group/project access token, api scope
flux check --pre
flux bootstrap gitlab --owner=$ROOT/domain-shared/configurations --repository=flux-releases \
  --branch=main --path=clusters/np --token-auth --personal=false
flux get kustomizations -A
```
Repo layout:
```
flux-releases/
├─ clusters/np/
│  ├─ flux-system/                 (generated)
│  ├─ 10-infra-controllers.yaml    Kustomization -> infrastructure/controllers
│  ├─ 20-infra-configs.yaml        dependsOn controllers -> infrastructure/configs
│  └─ 30-apps.yaml                 dependsOn configs     -> apps/np
├─ infrastructure/
│  ├─ controllers/  sources.yaml  istio-base.yaml  istiod.yaml
│  └─ configs/      namespaces.yaml  mesh-policy.yaml  netpol-*.yaml  quotas.yaml
└─ apps/np/{orders,payments,inventory}/helmrelease.yaml
```
Implementation order and **why**:

| Order | Object | Why before the next |
|---|---|---|
| 1 | HelmRepository `istio`, OCI HelmRepository `acme-charts` | Sources must exist before releases |
| 2 | `istio-base` (CRDs) | CRDs before anything that uses them |
| 3 | `istiod` (`dependsOn: istio-base`) | Webhook must be healthy **before** pods are created or sidecars silently don't inject |
| 4 | Namespaces with `istio-injection: enabled` | Label must exist **before** workload pods start |
| 5 | Mesh policies: `PeerAuthentication` STRICT, default-deny NetworkPolicies | Security baseline before apps exist ("secure by default") |
| 6 | Apps | Last |

```yaml
# clusters/np/10-infra-controllers.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: { name: infra-controllers, namespace: flux-system }
spec:
  interval: 10m
  path: ./infrastructure/controllers
  prune: true
  wait: true            # block dependants until Ready
  timeout: 10m
  sourceRef: { kind: GitRepository, name: flux-system }
---
# 20-infra-configs.yaml: same shape, name infra-configs, path ./infrastructure/configs, + spec.dependsOn: [{name: infra-controllers}]
# 30-apps.yaml:        name apps, path ./apps/np, + dependsOn: [{name: infra-configs}]
```
```yaml
# infrastructure/controllers/istiod.yaml  (lean for e2-standard-2 nodes)
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata: { name: istiod, namespace: istio-system }
spec:
  interval: 10m
  dependsOn: [{ name: istio-base }]
  chart: { spec: { chart: istiod, sourceRef: { kind: HelmRepository, name: istio, namespace: flux-system } } }
  values:
    global:
      hub: us-central1-docker.pkg.dev/my-devops-journey-502420/dockerhub-mirror/istio   # pulls via Door 1
      proxy: { resources: { requests: { cpu: 50m, memory: 96Mi }, limits: { memory: 256Mi } } }
    pilot: { resources: { requests: { cpu: 200m, memory: 512Mi } } }
    meshConfig:
      accessLogFile: /dev/stdout
      accessLogEncoding: JSON                        # Envoy access logs → stdout → Cloud Logging
      outboundTrafficPolicy: { mode: ALLOW_ANY }     # → REGISTRY_ONLY in S5
```
Hints: HelmRepository `url: https://istio-release.storage.googleapis.com/charts`. If `istiod` fails to create pods, check the webhook firewall rule from P3 (`15017` from the master CIDR). Resource budget: istiod ≈ 200m/512Mi, each pod ≈ app 100m + proxy 50m + otel 50m + fluent-bit 25m — start with **1 replica per service**.
**Verify:** `flux get all -A`, `kubectl -n istio-system get pods`, `istioctl analyze -A`.
**Gate G5:** delete `istiod`'s HelmRelease, create a test pod in a labelled namespace → no sidecar. Re-apply → still no sidecar until pod restarts. Explain why.

---

## P6 — Tier-1 `observability` (central logging) + sidecar configs

```hcl
# platform/observability/main.tf
variable "project_id" {} 
variable "region" { default = "us-central1" }
locals { domains = ["orders", "payments", "inventory"] }

resource "google_logging_project_bucket_config" "central" {
  project = var.project_id  location = var.region  bucket_id = "central-app-logs"
  retention_days = 30  enable_analytics = true            # Log Analytics (SQL over logs)
}
resource "google_logging_project_sink" "domains" {
  name = "domains-to-central"
  destination = "logging.googleapis.com/${google_logging_project_bucket_config.central.id}"
  filter = "resource.type=\"k8s_container\" AND resource.labels.namespace_name=~\"^dom-\""
  unique_writer_identity = true
}
# avoid double-ingesting into _Default:
resource "google_logging_project_exclusion" "dom_default" {
  name = "exclude-dom-from-default"
  filter = "resource.type=\"k8s_container\" AND resource.labels.namespace_name=~\"^dom-\""
}   # NOTE: exclusions on _Default are per-sink; manage via google_logging_project_sink for "_Default" in real use
resource "google_logging_metric" "errors" {
  name = "domain_errors"
  filter = "resource.type=\"k8s_container\" AND severity>=ERROR AND resource.labels.namespace_name=~\"^dom-\""
  metric_descriptor {
    metric_kind = "DELTA" value_type = "INT64"
    labels { key = "domain" value_type = "STRING" }
  }
  label_extractors = { "domain" = "EXTRACT(resource.labels.namespace_name)" }
}
```
> The exclusion block above is simplified: in a real setup you modify the built-in `_Default` sink's exclusions. Check the provider docs for `google_logging_project_sink` with name `_Default`. Treat this as a documented TODO to fix yourself.

**OTel Collector sidecar config** (ConfigMap in the chart; verify exporter option names against current `googlecloud` exporter docs):
```yaml
receivers:
  otlp: { protocols: { grpc: { endpoint: 127.0.0.1:4317 }, http: { endpoint: 127.0.0.1:4318 } } }
processors:
  resourcedetection: { detectors: [gcp], timeout: 5s }
  batch: {}
exporters:
  googlecloud:
    log: { default_log_name: "app-otel" }
service:
  pipelines:
    logs:    { receivers: [otlp], processors: [resourcedetection, batch], exporters: [googlecloud] }
    traces:  { receivers: [otlp], processors: [resourcedetection, batch], exporters: [googlecloud] }
```
**Fluent Bit sidecar (file-tail pattern)** — option names depend on version, confirm in the Fluent Bit docs:
```ini
[SERVICE]
    Flush 5
    Parsers_File parsers.conf
[INPUT]
    Name tail
    Path /var/log/app/*.log
    Parser json
    Tag app.${DOMAIN}
[FILTER]
    Name modify
    Match *
    Add domain ${DOMAIN}
[OUTPUT]
    Name stackdriver
    Match *
    resource k8s_container
    k8s_cluster_name acme-np
    k8s_cluster_location us-central1-a
    namespace_name ${NAMESPACE}
    pod_name ${POD_NAME}
    container_name app
    export_to_project_id ${PROJECT_ID}
```
**Verify (after P7 has pods):**
```bash
gcloud logging read 'resource.type="k8s_container" resource.labels.namespace_name="dom-orders"' --limit 5 --format json
gcloud logging read 'logName:"app-otel"' --limit 3
gcloud logging buckets describe central-app-logs --location us-central1
```
**Gate G6:** show one request's trail in Logs Explorer: app JSON log + Envoy access log + OTel/Fluent Bit record, correlated by trace/request id.

---

## P7 — Tier-3 three domains (infra + app + chart)

### 7a. Shared Helm chart (Tier 2 `helm-charts/acme-microservice`)
Values you want the chart to expose: `image.repository/tag`, `port`, `env`, `domain`, `serviceAccount.gsaEmail`, `sidecars.otel.enabled`, `sidecars.fluentbit.enabled`, `resources`. Key template fragment:

```yaml
# templates/deployment.yaml (excerpt)
spec:
  template:
    metadata:
      labels: { app: {{ .Release.Name }}, domain: {{ .Values.domain }} }
    spec:
      serviceAccountName: {{ .Release.Name }}
      volumes:
        - { name: applogs, emptyDir: {} }
        - { name: otel-config, configMap: { name: {{ .Release.Name }}-otel } }
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports: [{ containerPort: {{ .Values.port }} }]
          env: {{- toYaml .Values.env | nindent 12 }}
          volumeMounts: [{ name: applogs, mountPath: /var/log/app }]
        {{- if .Values.sidecars.otel.enabled }}
        - name: otel-collector
          image: {{ .Values.sidecars.otel.image }}     # from the AR mirror
          args: ["--config=/conf/config.yaml"]
          volumeMounts: [{ name: otel-config, mountPath: /conf }]
          resources: { requests: { cpu: 50m, memory: 64Mi }, limits: { memory: 128Mi } }
        {{- end }}
        {{- if .Values.sidecars.fluentbit.enabled }}
        - name: fluent-bit
          image: {{ .Values.sidecars.fluentbit.image }}
          volumeMounts: [{ name: applogs, mountPath: /var/log/app, readOnly: true }]
          resources: { requests: { cpu: 25m, memory: 32Mi }, limits: { memory: 64Mi } }
        {{- end }}
```
ServiceAccount template annotation: `iam.gke.io/gcp-service-account: {{ .Values.serviceAccount.gsaEmail }}`.
Publish: `helm package . && helm push acme-microservice-0.1.0.tgz oci://us-central1-docker.pkg.dev/<proj>/apps` (or the GitLab OCI registry) — Flux uses an `OCI` `HelmRepository`.

### 7b. Tier-3 `<domain>-infra` (Terraform), using the Tier-1 module
```hcl
# platform/terraform-modules/modules/app-identity/main.tf
variable "project_id" {}
variable "name" {}                 # GSA account id, e.g. orders-app
variable "namespace" {}
variable "ksa_name" {}
variable "project_roles" { type = list(string) default = [] }
resource "google_service_account" "app" { project = var.project_id account_id = var.name }
resource "google_service_account_iam_member" "wi" {
  service_account_id = google_service_account.app.name
  role   = "roles/iam.workloadIdentityUser"
  member = "serviceAccount:${var.project_id}.svc.id.goog[${var.namespace}/${var.ksa_name}]"
}
resource "google_project_iam_member" "roles" {
  for_each = toset(var.project_roles)
  project = var.project_id role = each.value
  member = "serviceAccount:${google_service_account.app.email}"
}
output "email" { value = google_service_account.app.email }
```
```hcl
# applications/orders/orders-infra/main.tf
module "identity" {
  source        = "git::https://gitlab.com/acme-labs-yourhandle/platform/terraform-modules.git//modules/app-identity?ref=v0.1.0"
  project_id    = var.project_id
  name          = "orders-app"
  namespace     = "dom-orders"
  ksa_name      = "orders"
  project_roles = ["roles/logging.logWriter", "roles/cloudtrace.agent", "roles/monitoring.metricWriter"]
}
```
Repeat for payments/inventory (copy = **three lines of difference**, which is the whole point of Tier 1 modules).

### 7c. Minimal services (different languages, same contract: `/healthz`, `/work`; JSON logs with `severity`)
```go
// orders (Go)
package main
import ("encoding/json";"log/slog";"net/http";"os")
func main() {
  lg := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{ReplaceAttr: func(_ []string, a slog.Attr) slog.Attr {
    if a.Key == "level" { a.Key = "severity" }; if a.Key == "msg" { a.Key = "message" }; return a }}))
  call := func(u string) int { r, err := http.Get(u); if err != nil { lg.Error("call failed", "url", u, "err", err.Error()); return 0 }; defer r.Body.Close(); return r.StatusCode }
  http.HandleFunc("/healthz", func(w http.ResponseWriter, _ *http.Request) { w.Write([]byte("ok")) })
  http.HandleFunc("/work", func(w http.ResponseWriter, _ *http.Request) {
    p := call(os.Getenv("PAYMENTS_URL") + "/work"); i := call(os.Getenv("INVENTORY_URL") + "/work")
    lg.Info("order processed", "domain", "orders", "payments", p, "inventory", i)
    json.NewEncoder(w).Encode(map[string]int{"payments": p, "inventory": i}) })
  http.ListenAndServe(":8080", nil)
}
```
```python
# payments (Python / FastAPI)   pip: fastapi uvicorn
import json, logging, sys
from fastapi import FastAPI
class J(logging.Formatter):
    def format(self, r): return json.dumps({"severity": r.levelname, "message": r.getMessage(), "domain": "payments"})
h = logging.StreamHandler(sys.stdout); h.setFormatter(J()); log = logging.getLogger("p"); log.addHandler(h); log.setLevel("INFO")
app = FastAPI()
@app.get("/healthz")
def hz(): return "ok"
@app.get("/work")
def work(): log.info("payment authorised"); return {"ok": True}
# run: uvicorn main:app --host 0.0.0.0 --port 8080
```
```js
// inventory (Node / Express)   npm i express
const app = require('express')();
const log = (severity, message) => console.log(JSON.stringify({ severity, message, domain: 'inventory' }));
app.get('/healthz', (_, r) => r.send('ok'));
app.get('/work', (_, r) => { log('INFO', 'stock reserved'); r.json({ ok: true }); });
app.listen(8080);
```
Call graph: **orders → payments, orders → inventory**; payments ↔ inventory **denied**. Env: `PAYMENTS_URL=http://payments.dom-payments.svc.cluster.local`, `INVENTORY_URL=http://inventory.dom-inventory.svc.cluster.local`.

### 7d. App CI (Tier 3): build → push to Artifact Registry → bump tag in `flux-releases`
- Image build on GitLab shared runners with Docker-in-Docker; authenticate to AR using WIF (same `id_tokens` pattern as P2, SA = a push-only GSA created in `bootstrap`/`gke-platform` with `roles/artifactregistry.writer`; get a token with `gcloud auth print-access-token` and `docker login -u oauth2accesstoken`).
- Tag bump: a job commits `image.tag: $CI_COMMIT_SHORT_SHA` into `apps/np/<domain>/helmrelease.yaml` in `flux-releases`. (Stretch: replace with Flux Image Automation.)
- Remember the rule from §3: `AR_REPO` is defined only at `applications/`.

### 7e. Default-deny + explicit allows (Flux, `infrastructure/configs`)
```yaml
# per namespace, e.g. dom-payments
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: dom-payments }
spec: { podSelector: {}, policyTypes: [Ingress, Egress] }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-baseline-egress, namespace: dom-payments }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } } }]   # DNS
      ports: [{ port: 53, protocol: UDP }, { port: 53, protocol: TCP }]
    - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: istio-system } } }]   # istiod xDS
      ports: [{ port: 15012 }, { port: 15014 }]
    - to: [{ ipBlock: { cidr: 169.254.169.252/32 } }]                    # GKE metadata server (Workload Identity!)
      ports: [{ port: 988 }]
    - to: [{ ipBlock: { cidr: 0.0.0.0/0, except: [10.0.0.0/8] } }]       # Google APIs for sidecars
      ports: [{ port: 443 }]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-from-orders, namespace: dom-payments }
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: dom-orders } } }]
      ports: [{ port: 8080 }]
```
```yaml
# L7 identity rule (payments only accepts the orders ServiceAccount)
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: { name: only-orders, namespace: dom-payments }
spec:
  action: ALLOW
  rules:
    - from: [{ source: { principals: ["cluster.local/ns/dom-orders/sa/orders"] } }]
---
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default, namespace: dom-payments }
spec: { mtls: { mode: STRICT } }
```
**Verify matrix** (run all, compare with prediction):
```bash
kubectl -n dom-orders exec deploy/orders -c app -- wget -qO- payments.dom-payments/work     # allowed
kubectl -n dom-payments exec deploy/payments -c app -- wget -qO- -T3 inventory.dom-inventory/work   # blocked (timeout = netpol)
# temporarily delete the NetworkPolicy deny and retry -> now 403 RBAC from Envoy (= AuthorizationPolicy). Two layers, two failure signatures.
kubectl -n dom-orders exec deploy/orders -c app -- wget -qO- --header 'Host: x' http://169.254.169.252:988   # WI path alive? (expected only if allow rule present)
istioctl proxy-status; istioctl x describe pod <orders-pod> -n dom-orders
kubectl -n dom-orders get pod -l app=orders -o jsonpath='{.items[0].spec.containers[*].name}'   # app istio-proxy otel-collector fluent-bit
```
**Nuances revealed:** default-deny **egress** breaks Workload Identity (metadata server `169.254.169.252:988`) and DNS and xDS — you must allow them; NetworkPolicy matches the *original* destination port (sidecar redirect happens after); STRICT mTLS rejects plaintext from non-mesh callers (relevant to P9).
**Gate G7:** the call-graph holds, and you can state which layer produced each rejection from its error signature alone.

---

## P8 — Close the doors (delete the default route)

**Goal:** flip `training_wheels=false`, then repair each break. Keep a table; fill the "Door used" column from evidence (flow logs / connectivity tests), not memory.

| Flow | Expected door after P8 | Fix if broken |
|---|---|---|
| Node → `*.googleapis.com` (logging, AR, GCS) | 1: PGA + private VIP | `private-googleapis-vip` route + DNS zones (P3) |
| Node → `*.pkg.dev` image pulls | 1 | `pkgdev-private` zone |
| Flux source-controller → `gitlab.com` | 3: tagged nodes + NAT | node tag `allowed-open-nat` |
| Charts from `istio-release.storage.googleapis.com` | 1 (googleapis name) | DNS wildcard resolves to private VIP |
| Docker Hub pulls | avoid: use AR mirror | repoint image refs to mirror |
| Partner API requiring fixed IP | 2 or 3 with static NAT IPs | `vip_allowlist` entry |
| Egress from pods | masqueraded to node IP, rides node tag | per-pod control needs NetworkPolicy/Istio, not tags |

```bash
cd platform/network && terraform apply -var training_wheels=false
kubectl get nodes; kubectl -n flux-system get gitrepositories     # which break first?
gcloud compute routes list --filter="network:acme-main"
gcloud logging read 'resource.type="gce_subnetwork" jsonPayload.connection.dest_ip:"140."' --limit 5   # VPC flow logs
```
Extra exercises: (1) remove `allowed-open-nat` from the node pool tags → watch Flux go red; (2) block Door 3 but keep PGA → what still works? (3) set mesh `outboundTrafficPolicy: REGISTRY_ONLY` (S5) and add a `ServiceEntry` for the one external host you need.
**Gate G8:** you can draw the packet path for each row without looking, including where SNAT happens.

---

## P9 — Internal Gateway (proxy-only subnet in action)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: internal-gw, namespace: dom-orders }
spec:
  gatewayClassName: gke-l7-rilb               # regional internal ALB — needs the proxy-only subnet
  listeners: [{ name: http, port: 80, protocol: HTTP }]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: orders, namespace: dom-orders }
spec:
  parentRefs: [{ name: internal-gw }]
  rules: [{ backendRefs: [{ name: orders, port: 8080 }] }]
```
Test from the `probe` VM: `curl http://<gateway-ip>/work`.
**Expected breaks (diagnose each):** (1) no proxy-only subnet or firewall `allow-proxy-only-to-nodes` → 502/timeouts; (2) STRICT mTLS rejects the LB's plaintext → use a `portLevelMtls: { 8080: { mode: PERMISSIVE } }` exception or route via an Istio ingress gateway instead; (3) health-check source ranges (130.211.0.0/22, 35.191.0.0/16) blocked by default-deny NetworkPolicy.
**Gate G9:** you can explain what the proxy-only subnet IPs are *used for* and why your own VMs never get one.

---

## P10 — Stretch menu (pick by interest; each is self-contained)

| ID | Lab | Nuance it reveals |
|---|---|---|
| S1 | **Private Services Access + Cloud SQL** in `shared-services`: `google_compute_global_address` (purpose `VPC_PEERING`, `100.64.0.0/20`) + `google_service_networking_connection`; add Cloud SQL Auth Proxy as a 5th sidecar | PSA is a *peering*: non-transitive; custom-route export needed for hybrid |
| S2 | **Hybrid Door 4**: second VPC `onprem-sim` (`192.168.100.0/24`), HA VPN + Cloud Router BGP both ways (hourly cost — tear down same day); DNS peering/forwarding; `ip-masq-agent` non-masquerade CIDRs | Why pod IPs are unknown to "on-prem"; overlapping CIDR pain |
| S3 | **Per-domain node pools** each with its own pod secondary range, node SA and taint; VPC firewall rules by source range/SA | Real VPC-level per-domain policy inside one cluster |
| S4 | **Consul contract bus** on free-tier e2-micro + GitLab runner on the same VM; swap `terraform_remote_state` for the Consul provider | Private services vs SaaS runners; Door 3 for the runner |
| S5 | Mesh `REGISTRY_ONLY` egress + `ServiceEntry` | Pod-level egress control (tags are node-level) |
| S6 | **Shared VPC**: host project owns network, service project owns cluster | Cross-project IAM (`compute.networkUser`, GKE service agents) |
| S7 | Add **Cloud Storage / Firebase / Pub-Sub** the Tier-1-module way (below) | How the platform grows without copy-paste |

**S7 pattern — extending infra later** (this is why modules live in Tier 1):
```hcl
# Tier 1: platform/terraform-modules/modules/gcs-bucket  (uniform access, PAP enforced, labels, optional IAM members)
# Tier 3: applications/orders/orders-infra
module "uploads" {
  source = "git::https://gitlab.com/$ROOT/platform/terraform-modules.git//modules/gcs-bucket?ref=v0.2.0"
  name   = "${var.project_id}-orders-uploads"
  writers = ["serviceAccount:${module.identity.email}"]
}
```
Then expose the bucket name to the chart via Flux `postBuild.substitute` / HelmRelease `values` — **by deterministic naming**, no lookup service needed.

---

## 5. Cost + teardown

Rough order of magnitude while up (verify current pricing; use the budget alarm): 2–3 spot `e2-standard-2`, one Cloud NAT gateway, 2 in-use static IPs, one internal LB, small PDs → **a few dollars per day**; near-zero when destroyed (keep state buckets). Cloud SQL, HA VPN tunnels and external LBs are the expensive stretch items — run them for hours, not days.

Teardown order (reverse of build) — script it as `destroy-all.sh` like your earlier ones:
1. `flux suspend` apps → delete `Gateway` objects (so LB resources are removed before the VPC) 
2. `gke-platform` → `network` (NAT/router/addresses) → keep `bootstrap`
3. Orphans to check: `gcloud compute forwarding-rules list`, `addresses list`, `disks list`, `routers list`.

---

## 6. Hidden-nuance checklist (tick when you've *observed* it, not read it)

- [ ] Cloud NAT needs a default-internet-gateway next hop
- [ ] Private Google Access vs private.googleapis.com VIP (routes + DNS both required)
- [ ] Tags/SAs on routes & firewalls are node-level; pods inherit
- [ ] Private cluster: master → node webhook ports (15017/8443/9443)
- [ ] Pod/service secondary-range sizing and `max_pods_per_node`
- [ ] Proxy-only subnet purpose + firewall from it to backends
- [ ] LB health-check ranges vs default-deny policies
- [ ] Default-deny egress breaks DNS, xDS and Workload Identity metadata server
- [ ] STRICT mTLS vs plaintext callers (LB, kubelet probes handled by Istio rewrite)
- [ ] NAT port exhaustion / Docker Hub rate limits → mirror
- [ ] PSA is non-transitive peering
- [ ] `delete_default_routes_on_create` is create-time only
- [ ] WIF attribute conditions as the real trust boundary

---

## 7. Hand-offs to the other personas

- **Tutor** first topics (only if you want to be tested): "Which door does this packet use?" and "Why did the pod not get a sidecar?"
- **Editor** first artifact: your `network` stack at the end of P3 — ask it to *defend* each CIDR and each firewall rule.
- **Roommate**: pair P8 with a "city road network / customs border" analogy session.
