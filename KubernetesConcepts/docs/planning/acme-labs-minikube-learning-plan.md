# Acme Labs — Minikube Learning Plan (incremental, network-first)

**Purpose:** a systematic, stage-by-stage plan for learning the *details* of Kubernetes networking and platform plumbing on **minikube**, so that GCP (plan P3–P9) becomes a rehearsed repetition with only the cloud-specific parts new.
**This file is a plan, not a lab manual.** Each stage says what to learn, build, break, capture and prove. Command-level labs already exist in `acme-labs-k8s-on-docker-practice.md` (kind-based, K1–K10); the translation table in §3 maps them to minikube. Fine-tune this plan first (§12), then we expand one stage at a time.

---

## 1. Quick answer: can Kubernetes create a VPC or handle subnet-to-subnet traffic?

| Question | Answer |
|---|---|
| Can Kubernetes create a VPC / subnet / route / NAT / firewall? | **No.** Those belong to the layer *below* it (GCP VPC via Terraform; here: Docker networks + a router container). |
| Can something running *in* Kubernetes create them? | Only by calling the cloud API (Config Connector, Crossplane, Flux tf-controller). The cloud still creates the object. Not the goal here: you want to hand-write the network. |
| Does Kubernetes know about subnets? | No. It knows **nodes**, a **pod CIDR**, a **service CIDR**. Which subnet a node is in is invisible to it. |
| How do pods in different subnets talk? | Kubernetes *assumes* a flat pod network; the **CNI** (Cilium) delivers it. On GCP the VPC routes pod IPs natively (alias-IP secondary ranges). On minikube the CNI tunnels/encapsulates. |
| Who controls what may talk to what? | In-cluster: **NetworkPolicy** + mesh policy. Between subnets/VPCs: **routes + firewall rules** (cloud layer). Two layers, two owners, two failure signatures. |
| What does Kubernetes ask the cloud for? | `Service type=LoadBalancer` → cloud LB; GKE auto-creates some firewall rules/routes. |

So **on minikube you simulate the layer below Kubernetes yourself** (Docker network = VPC, router container = routes/firewall, `ip route` on nodes = doors). That is exactly where the "hidden nuances" live.

---

## 2. How this plan is designed (principles you can veto)

1. **Network onion, inside-out in dependency order:** address plan → node/subnet networking → pod network (CNI) → services/DNS → workloads → egress → policy → identity → observability → entry → GitOps → integration. One layer per stage; each layer only makes sense once the one under it is understood.
2. **One new concept per stage, one new failure family per stage.** (Breadth over depth; depth comes later per stage when you ask.)
3. **Teach first, then do, then break.** Every stage opens with a "Learn first" list (mechanics only — I'll explain any item on request, directly, not by quizzing). Predictions come *after* the mechanics, to check understanding, not to test cold.
4. **Evidence over belief.** Every claim ("pods are SNAT'd to the node IP") must be backed by a command output saved in your journal.
5. **Break it on purpose** — each stage has 3–4 deliberate failures; you record the *symptom → layer → proof → fix*. This is how "attention to detail" is trained.
6. **Regression safety:** every stage ships `verify-sN.sh`; `verify-all.sh` must stay green as layers are added.
7. **Rebuild-from-Git is the end state** (S11–S12): the cluster is disposable; knowledge lives in the repo and notebooks.
8. **Pace by gates, not calendar.** Effort estimates below are indicative only.

---

## 3. Minikube specifics (read before S1)

### 3a. Flags and what I verified (against minikube docs, Oct 2026)
| Need | Minikube option | Status |
|---|---|---|
| Choose node subnet | `--subnet=10.10.0.0/24` (docker/podman driver; default starts at 192.168.49.0) | ✔ documented |
| Use an existing Docker network | `--network=<name>` | ✔ documented |
| Fixed node IP | `--static-ip=10.10.0.10` (private IPv4, last octet 2–254; **cannot add to an existing cluster**) | ✔ documented |
| Service CIDR | `--service-cluster-ip-range=10.30.0.0/24` (default `10.96.0.0/12`) | ✔ documented |
| Pod CIDR | `--extra-config=kubeadm.pod-network-cidr=10.20.0.0/20` | ⚠ **not in the docs I read — verify in S1** (check `node.spec.podCIDR` and actual pod IPs) |
| CNI | `--cni=auto\|bridge\|calico\|cilium\|flannel\|kindnet\|<manifest path>`; default kindnet **does not enforce NetworkPolicy**; docs show Calico for policy | ✔ documented; ⚠ no documented "no CNI" option → start with built-in `cilium`, and check who allocates pod IPs (S1/S3) |
| Multi-node | `--nodes=N` or `minikube node add` (default 1) | ✔ documented |
| CPU / RAM | `--cpus`, `--memory` (docker driver: per node container) | ✔ documented |
| Runtime | `--container-runtime=docker\|containerd\|cri-o` → choose **containerd** (GKE uses containerd) | ✔ documented |
| Load local images | `minikube image load <img> -p acme` (replaces `kind load`) | ✔ standard |
| Isolation per experiment | `-p <profile>`, `minikube profile list`, `minikube delete -p <p>` | ✔ standard |
| Ingress addons | `ingress` / `ingress-dns` addons are **Linux-only** on the docker driver | ✔ documented |
| LoadBalancer on non-Linux | `minikube tunnel`; NodePort via `minikube service <svc> --url` | ✔ documented (node IP not reachable from host on Windows/WSL2; assume macOS too and test) |

> Run `minikube start --help` once and confirm every flag above on **your** version before building on it.

### 3b. Translation from the kind lab file
| In `acme-labs-k8s-on-docker-practice.md` | On minikube |
|---|---|
| `kind create cluster --config …` | `minikube start -p acme --driver=docker --container-runtime=containerd --nodes=2 --subnet=10.10.0.0/24 --service-cluster-ip-range=10.30.0.0/24 --extra-config=kubeadm.pod-network-cidr=10.20.0.0/20 --cni=cilium` (verify as above) |
| Nodes `acme-control-plane`, `acme-worker`, `acme-worker2` | `acme`, `acme-m02`, `acme-m03` (confirm: `minikube node list -p acme`); they are containers → `docker exec <node> …` or `minikube ssh -p acme -n <node>` |
| `docker network create acme-main` | minikube creates the network itself (named after the profile — confirm with `docker network ls`) **or** pre-create and pass `--network=` |
| Second subnet + router + probe containers (K3) | identical (Docker commands) — attach to the minikube network |
| `kind load docker-image` | `minikube image load -p acme <image>` |
| MetalLB manifest (K9) | `minikube addons enable metallb -p acme` + `minikube addons configure metallb`, **or** the manifest/Helm chart (needed later for Flux) |
| `kind delete cluster` | `minikube delete -p acme` |

### 3c. OS caveats (tell me yours — §12)
- **Linux:** node IPs reachable from the host; ingress addon works.
- **macOS / Windows (Docker Desktop):** run network tests from **containers** (`probe`) or `kubectl exec`; use `minikube tunnel` for LoadBalancer; ingress addon not supported on docker driver.
- All: node-container routes are **lost on restart** → keep `scripts/routes.sh`.

---

## 4. Reference topology and how it grows

Final local topology (mirrors the GCP plan §2):

```
 acme-ops 10.10.1.0/28                         acme (minikube) 10.10.0.0/24
 [probe 10.10.1.3] ─ 10.10.1.2 [vpc-router] 10.10.0.250 ─ [acme][acme-m02][acme-m03]
                                  (routes + firewall)        pods 10.20.0.0/20 · svcs 10.30.0.0/24
 inside the cluster:  dom-orders | dom-payments | dom-inventory | observability | istio-system | flux-system
```
| After stage | Topology gains |
|---|---|
| S1 | 1–2 nodes, explicit node/pod/service ranges |
| S2 | ops subnet, router, probe, firewall |
| S3 | Cilium + Hubble visibility |
| S4 | Services, DNS behaviour understood |
| S5 | three real services + Helm chart |
| S6 | egress doors (default route removed, 3rd node as "tagged") |
| S7 | NetworkPolicy default-deny + call graph |
| S8 | Istio sidecars, mTLS, AuthorizationPolicy |
| S9 | 4-container pod + central log aggregator |
| S10 | LoadBalancer/Ingress entry from the ops subnet |
| S11 | everything owned by Flux/Git, ordered |
| S12 | cold-rebuild proof + GCP delta |

---

## 5. The learning system (what makes it incremental)

### 5a. Lab repo layout
```
acme-k8s-lab/
  README.md                    # the stage checklist (tick as gates pass)
  notebook/
    ipam.md                    # address table, grows each stage
    packet-paths.md            # "src → hops → dst" with the command that proved each hop
    failure-catalog.md         # symptom | layer | how to confirm | fix
    decisions.md               # why this choice (CNI, mesh, ranges …)
    predictions.md             # prediction vs result (after reading the mechanics)
  scripts/
    cluster-up.sh  cluster-down.sh  routes.sh
    verify-s01.sh … verify-s12.sh  verify-all.sh   reset-sNN.sh
  stages/sNN-<name>/           # README (goal, tasks, evidence), manifests/
  apps/ orders payments inventory
  charts/acme-microservice
  gitops/                      # later mirrored into the Flux repo
```

### 5b. Templates (copy into notebook/)
- **IPAM row:** `name | CIDR | owner (who allocates) | routable beyond cluster? | overlap risk | GCP twin`
- **Packet path:** `flow | hop 1 … hop n | NAT/SNAT where? | policy/firewall that decides | proof command + output`
- **Failure signature:** `symptom (exact text) | layer | first command to confirm | root cause | fix | stage first seen`
- **Prediction:** `question | my prediction | actual | why I was right/wrong`

### 5c. Loop for every lab (≈ the unit of progress)
1. **Learn first** (20–30 min): the stage's mechanics list.
2. **Predict:** write 3–5 expectations in `predictions.md`.
3. **Do:** build the stage increment; script it (`stages/sNN/…`).
4. **Observe:** capture evidence into the journal.
5. **Break:** run each deliberate failure; add to the failure catalog.
6. **Explain:** 3 lines in your own words + a hand-drawn diagram.
7. **Gate:** tick the checklist; `verify-sNN.sh` and `verify-all.sh` green.

### 5d. Review cadence
- **End of each Part (A–E):** redraw the whole topology from a blank page; re-derive the IPAM table; re-run `verify-all.sh` on a fresh cluster.
- **Failure-catalog drill:** pick 5 random symptoms, name the layer without running anything.
- **Teach-back:** 10-minute spoken walkthrough of one packet path per Part.

---

## 6. Stage overview

| # | Stage | Layer | New concept | New failure family | Kind lab | GCP twin | Effort* |
|---|---|---|---|---|---|---|---|
| S0 | Workbench + defaults | — | what minikube gives you by default | restart behaviour | — | P0 | S |
| S1 | Address plan + cluster | L0 address space | node/pod/service CIDRs, IPAM | silent overlaps, wrong IPAM owner | K1, K2 | P3, P4 | M |
| S2 | Below Kubernetes: subnet, router, firewall | L2/L3 | next hop, ARP, conntrack | asymmetric routing, stateful firewall drops | K3 | P3 | M–L |
| S3 | Pod network (CNI) | pod net | veth, overlay vs native, masquerade | MTU/encap, agent down | — | P4 | M |
| S4 | Services, kube-proxy, DNS | virtual IPs | DNAT, endpoints, CoreDNS, ndots | no endpoints, port/targetPort | — | P7 | M |
| S5 | Three services + Helm chart | workloads | probes, requests/limits, chart values | Pending/OOM/ImagePull | K5 | P7 | M–L |
| S6 | Egress and doors | egress | default route, SNAT, node-level egress | DNS ok / TCP dead, image pulls | K4 | P3, P8 | M |
| S7 | NetworkPolicy | L3/L4 policy | default-deny, selectors, pod ports | timeouts, DNS blocked | K6 | P7 | M |
| S8 | Mesh identity | L7 identity | sidecar injection, mTLS, authz | 1/2 Ready, reset, 403 | K7 | P5, P7 | L |
| S9 | Sidecars + central logs | observability | sidecar patterns, log paths | silent log loss | K8 | P6 | M–L |
| S10 | Entry: LB/Ingress/Gateway | north-south | LB layers, client IP, health checks | timeout/reset/403 cascade | K9 | P9 | M |
| S11 | GitOps ordering (Flux) | delivery | dependsOn, wait, prune, drift | CRD-before-CR, no sidecar | K10 | P5 | L |
| S12 | Integration + GCP handoff | all | rebuild-from-Git, delta analysis | — | — | pre-flight P4–P9 | M |

*S ≈ 2 h, M ≈ 3–5 h, L ≈ 5–7 h. Whole plan ≈ 50–60 h. Pace by gates.

**Parts:** A = S0–S2 (address space + layer below k8s) · B = S3–S5 (inside the cluster) · C = S6–S8 (control traffic) · D = S9–S11 (observe, enter, deliver) · E = S12.

---

## 7. Stage details

> Format per stage: **Goal · Learn first · Build · Break · Capture · Gate · GCP link.** Keep what you like; cut or reorder in §12.

### S0 — Workbench + defaults (Part A)
- **Goal:** repo, notebooks and scripts exist; you have seen what minikube does *by default* before designing anything.
- **Learn first:** minikube on docker driver = node is a container; profiles; addons; where state lives; how `stop/start` differs from `delete`.
- **Build:** tool/version check (`docker info` memory, `minikube version`, `helm`, `flux`, `istioctl`, `kubectl`); repo skeleton + templates; a throwaway profile `scratch` started with defaults.
- **Inspect (fill a "Defaults sheet"):** node IP + docker network + subnet, `node.spec.podCIDR`, service CIDR, default CNI, runtime, kube-proxy mode, CoreDNS IP.
- **Break:** `minikube stop && start` → what changed/persisted (IP, routes you added by hand)?
- **Capture:** defaults vs your target ranges (do defaults collide with anything you plan?).
- **Gate:** you can state every default range and who allocated it.
- **GCP link:** P0 hygiene (budget alarm has no local twin — make one for your *laptop RAM* instead: `docker stats` baseline).

### S1 — Address plan + cluster (IPAM)
- **Goal:** design the address space on paper first; create the cluster to match; prove it.
- **Learn first:** CIDR arithmetic; three separate address spaces (node, pod, service); per-node pod slice (e.g. /24 per node out of a /20 → 16 nodes max); who allocates pod IPs (kube-controller-manager `node.spec.podCIDR` vs the CNI's own pool); what is routable outside the cluster (pods/services: **no**); RFC1918 vs 100.64/10; why GKE needs secondary ranges and a /28 master range.
- **Build:** 10 CIDR drills (capacity, overlap, split); IPAM v1 using the same numbers as GCP plan §2; start `acme` with explicit ranges (§3b); verify with `kubectl`/`docker network inspect`.
- **Open item to resolve:** does `--extra-config=kubeadm.pod-network-cidr` + built-in Cilium actually put pods in `10.20.0.0/20`? If not, find out *why* (IPAM mode lesson) and decide: Cilium via Helm with `ipam.mode=kubernetes` vs accept cluster-pool. Record in `decisions.md`.
- **Break:** start a 2nd profile with the **same pod CIDR** → both run happily (overlap is silent until you connect them, S2). Then make the service CIDR tiny and note how capacity fails.
- **Capture:** IPAM table v1 with evidence column; `decisions.md` entry.
- **Gate:** derive max nodes/pods from the CIDRs by hand and match `kubectl` output.
- **GCP link:** P3 subnets + secondary ranges, P4 `ip_allocation_policy`, `max_pods_per_node`.

### S2 — Below Kubernetes: second subnet, router, firewall
- **Goal:** reproduce "routes + next hop + firewall" which Kubernetes cannot see.
- **Learn first:** L2 vs L3; ARP; longest-prefix match; next hop; `ip_forward`; stateful firewall (conntrack, ESTABLISHED/RELATED); asymmetric routing; Docker bridge ≈ subnet, why two bridges don't talk by default.
- **Build:** `acme-ops` network, `vpc-router` (dual-homed, NET_ADMIN), `probe`; routes both ways; `FORWARD DROP` + explicit allows; reach the API server only from `probe` (analog of authorized networks); `routes.sh`.
- **Break:** remove return route; remove ESTABLISHED rule; give ops subnet an overlapping CIDR; restart a node and lose its routes.
- **Capture:** packet paths P1–P3 (with `tcpdump` on router), 4 failure signatures.
- **Gate:** predict each `curl` result across firewall changes before running.
- **GCP link:** P3 routes/firewall, IAP bastion, stretch S2 (hybrid).

### S3 — Pod network (CNI)
- **Goal:** know what actually happens when pod A talks to pod B.
- **Learn first:** pod netns + veth pair; CNI = IPAM + wiring; overlay (VXLAN/Geneve) vs native routing; where pod IPs get SNAT'd; kube-proxy vs eBPF datapath; identities; Hubble.
- **Build:** confirm Cilium healthy (`cilium status`); same-node and cross-node pod pings; `tcpdump` on a node's `eth0` to see encapsulation; inspect `ip route`, `ip link`, Cilium endpoint list; enable Hubble and watch flows.
- **Break:** delete the Cilium agent pod mid-traffic; disable masquerade and observe un-routable replies; (optional) MTU mismatch.
- **Capture:** pod→pod packet path (same node / different node); note "overlay here vs native on GCP".
- **Gate:** draw pod A (node 1) → pod B (node 2) with every hop and address rewrite.
- **GCP link:** P4 Dataplane V2 (Cilium) — same policy engine, different packet transport.

### S4 — Services, kube-proxy and DNS
- **Goal:** demystify "ClusterIP" and name resolution.
- **Learn first:** ClusterIP is virtual (DNAT, no interface); EndpointSlices; `port` vs `targetPort`; kube-proxy modes; NodePort; headless; CoreDNS, search domains, `ndots:5`, FQDN trailing dot; `externalTrafficPolicy`.
- **Build:** echo deployment + Service; read the DNAT rules on a node (`iptables-save | grep`); resolve from a pod with `dig` and watch query counts with different `ndots`; reach a NodePort from `probe` via the router.
- **Break:** selector mismatch (no endpoints), wrong targetPort, scale to 0, CoreDNS scaled to 0, cross-namespace short name.
- **Capture:** 5 failure signatures with the exact error text.
- **Gate:** explain where the destination IP changes (and where it doesn't) for a Service call.
- **GCP link:** all P7 service calls; GKE kube-dns / NodeLocal DNS.

### S5 — Three services + Helm chart
- **Goal:** realistic workloads to carry every later network experiment.
- **Learn first:** probes (liveness/readiness/startup), requests/limits vs allocatable (scheduling math), ServiceAccounts, env config, structured JSON logs (`severity`, `message`, `domain`), timeouts/retries between services, Helm templating (values, helpers, `lint`, `template`).
- **Build:** `orders` (Go), `payments` (Python), `inventory` (Node) with `/healthz` + `/work`; call graph orders→payments, orders→inventory; Dockerfiles; `minikube image load`; chart `acme-microservice` v1 (no sidecars yet); three releases in namespaces `dom-*`; `verify-s05.sh`.
- **Break:** failing readiness, OOMKilled, over-requested CPU (Pending), missing image (`ErrImageNeverPull`/`ImagePullBackOff`), wrong Service DNS name.
- **Capture:** capacity table (sum of requests vs node allocatable) — you will need it for sidecars in S8–S9.
- **Gate:** call graph works through DNS names; `helm lint` clean; resource table matches `kubectl describe node`.
- **GCP link:** P7a–c (sample code is in the GCP plan).

### S6 — Egress and the doors
- **Goal:** the P8 lesson in miniature — delete the default route, then earn every outbound flow back.
- **Learn first:** default route and longest prefix; SNAT/masquerade (who rewrites, where); DNS is resolved *out of band* from routing (name works, TCP dead); node-level egress (pods inherit it); why node labels are the local stand-in for tags; image-pull as an egress dependency.
- **Build:** add 3rd node; preload images; baseline source-IP observation (internal echo container logs client IP); remove default route on nodes; Door 2 (/32), Door 3 (default route only on the labelled node), Door 4 (route via router); fill the **door table from evidence**.
- **Break:** unlabel the node; delete a pod needing an un-preloaded image; mis-order routes.
- **Capture:** door table (flow → door → proof); packet-path entries for each door.
- **Gate:** draw the path and the SNAT point for each flow without notes.
- **GCP link:** P3 doors + P8. *Door 1 (Private Google Access + private DNS VIP) has no local equivalent — it is the first genuinely new GCP item.*

### S7 — NetworkPolicy
- **Goal:** enforce the call graph at L3/L4 and learn to read the verdicts.
- **Learn first:** additive allow model; default-deny; pod selectors vs namespace selectors; **ports are pod ports, not Service ports**; egress needs DNS; `ipBlock` semantics; how Cilium maps policy to identities; Hubble verdicts.
- **Build:** default-deny + DNS in each `dom-*`; allow orders→payments/inventory on pod port; verification matrix; watch `DROPPED` flows.
- **Break:** remove DNS allow; use Service port instead of pod port; mislabel a namespace; allow only ingress and forget egress.
- **Capture:** failure signatures: DNS-denied vs connection-denied vs refused.
- **Gate:** identify the failing layer from error text alone.
- **GCP link:** P7e; later NetworkPolicy must also allow the GKE metadata server (WI) — new on GCP.

### S8 — Mesh identity (Istio sidecars)
- **Goal:** identity-based control on top of network-based control.
- **Learn first:** injection mechanics (label → webhook → pod creation time; existing pods need restart); init container iptables redirect; Envoy bootstrap via xDS (istiod:15012); mTLS modes (PERMISSIVE/STRICT); SPIFFE identity = namespace + ServiceAccount; AuthorizationPolicy (any ALLOW policy ⇒ implicit deny for the workload); probe rewrite; interplay with NetworkPolicy.
- **Build:** install Istio (sidecar mode, minimal profile); label + restart; fix the egress-to-istiod break; STRICT in payments; plain-client test; "intruder" ServiceAccount in the same namespace; AuthorizationPolicy.
- **Break:** label after pods start (no sidecar); block istiod egress (pods stay 1/2); plaintext client vs STRICT; policy referencing wrong principal.
- **Capture:** three signatures: **timeout** (network), **reset** (mTLS), **403** (authz).
- **Gate:** name the layer for each without running a command.
- **GCP link:** P5 (webhook firewall rule from the master), P7.

### S9 — Sidecars and central logging
- **Goal:** four-container pod; all logs reach one central place.
- **Learn first:** sidecar pattern and its resource tax; shared `emptyDir`; three log paths (stdout via node agent, file-tail sidecar, OTLP sidecar); field conventions (`severity`, `message`, labels); central-sink concepts (what Cloud Logging's router/sink/bucket will do on GCP); `subPath` mounts.
- **Build:** log aggregator (Fluent Bit forward → stdout); pod with app + Fluent Bit + OTel collector (+ injected Envoy); chart toggles `sidecars.*`; NetworkPolicy holes for the aggregator; capacity re-check from S5.
- **Break:** wrong tail path; unreadable volume; aggregator down (buffering/backpressure); config mount hiding `parsers.conf`; mismatched field names.
- **Capture:** one request traced across app log, Envoy access log, OTel span, aggregator record.
- **Gate:** explain which log path needs which network hole.
- **GCP link:** P6 (Log Router sink, `central-app-logs` bucket, Workload Identity for sidecars).

### S10 — Entry: LoadBalancer / Ingress / Gateway
- **Goal:** traffic from outside the cluster, through every layer you built.
- **Learn first:** NodePort → LoadBalancer → Ingress → Gateway API layering; L4 vs L7 LB; client-IP preservation (`externalTrafficPolicy`); MetalLB L2 (ARP) mechanics; health checks; plaintext at the edge vs STRICT mTLS; on minikube: `tunnel`/`service --url` where the host can't reach node IPs; ingress addon is Linux-only on docker driver.
- **Build:** MetalLB pool inside the cluster subnet; LB service for `payments`; call it from `probe` via the router; then optional Gateway API (implementation decision in §12).
- **Break (chain of four):** router firewall → NetworkPolicy ingress → STRICT mTLS reset → authz 403 from a plaintext caller.
- **Capture:** the four-row break table (symptom → layer → fix).
- **Gate:** name the failing layer from the symptom alone.
- **GCP link:** P9 (proxy-only subnet, health-check ranges, internal ALB). Mechanics differ (managed Envoy outside your VPC); layers are the same.

### S11 — GitOps ordering (Flux)
- **Goal:** Git owns the cluster; ordering is explicit and provable.
- **Learn first:** Flux controllers; GitRepository / Kustomization / HelmRelease; `dependsOn` + `wait` + health checks; prune and drift; CRD-before-CR; namespace-label-before-pod; bootstrap chicken-and-egg (CNI must exist before Flux).
- **Build:** bootstrap against GitLab using the plan's group path; repo layout `controllers → configs → apps`; migrate S5–S10 manifests; delete manually-applied objects and let Flux recreate; image-tag bump flow (CI commit → reconcile).
- **Break:** remove `dependsOn`; label namespaces after workloads; drop `wait`; edit a live object (watch drift correction); suspend/resume.
- **Capture:** for every object: *why it is in controllers vs configs vs apps*.
- **Gate:** cold cluster + bootstrap ⇒ everything converges with zero manual `kubectl apply`.
- **GCP link:** P5 verbatim; the only change is Terraform creating the cluster underneath.

### S12 — Integration rehearsal + GCP handoff
- **Goal:** prove repeatability and know exactly what is *new* on GCP.
- **Do:** `minikube delete` → one script rebuilds cluster + routes + Cilium + Flux; `verify-all.sh` green; time it; write the runbook; produce the **GCP delta table** (§9) with a prediction for each row; 10-minute teach-back per Part.
- **Gate:** cold rebuild with no hand edits; delta table complete; you can name the first GCP phase to start and its expected failure points.

---

## 8. Cluster and resource strategy
| Stage | Nodes | Per-node (indicative) | Notes |
|---|---|---|---|
| S0–S1 | 1 (2 for the overlap break) | 2 CPU / 3 GB | defaults first |
| S2–S5 | 2 | 2 CPU / 3 GB | router/probe are tiny containers |
| S6–S7 | 3 | 2 CPU / 3 GB | 3rd node = "tagged" door |
| S8–S11 | 3 | 2 CPU / 4 GB | Istio + sidecars + Flux |
Host RAM: ≥ 12 GB workable, 16 GB comfortable. Verify with `docker stats` and the S5 capacity table. One profile (`acme`) is rebuilt between Parts; use extra profiles only for deliberate overlap/multi-cluster experiments. Use `minikube stop` between sessions; remember routes reset (S2).

---

## 9. Handoff to GCP — what carries over and what is new
| Topic | Local (minikube) | GCP | New learning there |
|---|---|---|---|
| Address plan | Docker subnet + flags | VPC subnet + secondary ranges + master /28 + proxy-only | Terraform for ranges; PSA (100.64/10) |
| Pod transport | CNI tunnels (overlay) | VPC-native alias IPs (native routing) | routes/firewall see pod CIDRs directly |
| Egress | `ip route` on nodes | Cloud NAT + routes + tags | tag-based routes; static NAT IPs |
| Google APIs | none | PGA + private DNS VIP + `199.36.153.8/30` | **first truly new item** |
| Identity to cloud | none | Workload Identity (KSA→GSA), WIF for CI | NetworkPolicy must allow metadata server |
| Firewall | iptables on router | VPC firewall (tags/SA/ranges) | master→node webhook ports |
| Entry | MetalLB / tunnel | managed internal ALB, proxy-only subnet | health-check source ranges |
| Logging | Fluent Bit aggregator | Cloud Logging API + Log Router sinks/buckets | sink filters, exclusions |
| Control plane | local | private endpoint + authorized networks | peering, /28 |
| Delivery | Flux on local cluster | same Flux, Terraform-built cluster | TF ↔ Flux handshake by names |

The Terraform / GitLab / WIF track (GCP P0–P2) is independent of this plan and can interleave.

---

## 10. "Details that bite" — tracker (tick when *observed*, with evidence link)
| Detail | Stage | Proof command idea |
|---|---|---|
| Pod CIDR slice per node limits node count | S1 | `kubectl get nodes -o custom-columns=…podCIDR` |
| Overlap is silent until you connect networks | S1→S2 | second profile + router |
| Routes on node containers vanish on restart | S0/S2 | `ip route` before/after |
| Return route / asymmetric routing | S2 | `tcpdump` on router |
| Stateful firewall needs ESTABLISHED | S2 | `iptables -L -v` counters |
| Pod IPs are SNAT'd leaving the cluster | S3/S6 | echo container logs |
| `port` vs `targetPort` | S4 | `kubectl get endpointslices` |
| `ndots:5` multiplies DNS queries | S4 | `dig +search` / tcpdump :53 |
| DNS works while TCP has no route | S6 | `dig` ok, `curl` timeout |
| Image pulls are an egress dependency | S6 | `ImagePullBackOff` after route removal |
| NetworkPolicy ports = pod ports | S7 | change 8080→80 |
| Default-deny egress breaks DNS | S7 | `bad address` vs timeout |
| Default-deny egress breaks mesh bootstrap | S8 | pods 1/2 Ready |
| Injection happens at pod creation | S8/S11 | missing sidecar |
| Any ALLOW AuthorizationPolicy ⇒ implicit deny | S8 | intruder gets 403 |
| STRICT mTLS rejects plaintext (LB, plain client) | S8/S10 | connection reset |
| `subPath` mount preserves sibling files | S9 | missing `parsers.conf` |
| Sidecars multiply resource requests | S5/S9 | Pending pods |
| Client IP lost without `externalTrafficPolicy: Local` | S10 | source IP in app/Envoy logs |
| CRD must exist before CR | S11 | "no matches for kind" |
| Namespace label before workloads | S11 | sidecar missing |

---

## 11. Risks and mitigations
| Risk | Mitigation |
|---|---|
| Laptop RAM exhausted at S8+ | per-stage resource table; stop idle stacks; 2-node fallback for S8–S9 |
| Built-in Cilium ignores your pod CIDR | S1 open item; decide Helm-installed Cilium |
| Docker Desktop networking quirks | run all tests from containers; `routes.sh` |
| Scope creep | each stage has a gate; extras go to "later" list in `decisions.md` |
| Forgetting | notebooks + failure catalog + review cadence (§5d) |

---

## 12. Fine-tuning panel (your decisions)
| Decision | Default in this plan | Alternatives | Effect |
|---|---|---|---|
| CNI | Cilium (matches GKE Dataplane V2) | Calico (documented in minikube), kindnet (no policy) | S3/S7 depth, Hubble |
| Mesh | Istio sidecar | Linkerd, Istio ambient (removes sidecars you want to study), skip mesh | S8–S10 |
| Runtime | containerd | docker | parity with GKE |
| Nodes | 2→3 | single node | S6 needs ≥2 |
| Entry | MetalLB + (optional) Gateway API | ingress-nginx addon (Linux only) | S10 |
| Git for Flux | gitlab.com group layout from GCP plan | local Gitea | S11 realism vs offline |
| Logging backend | Fluent Bit aggregator | Loki + Grafana | S9 weight |
| Services | Go/Python/Node as in GCP plan | one language | S5 effort |
| Stage order | as above | move S5 earlier/later; merge S3+S4 | cognitive load |
| Depth control | breadth first; deep-dive per stage on request | add a "deep" column | time |

**Questions for you (answer any, ignore the rest):**
1. Which OS / how much RAM? (decides tunnel vs direct IP, node count)
2. Keep Istio, or prefer to start without a mesh and add it as an optional S8?
3. Flux against gitlab.com (needs internet/token) or fully local?
4. Which stages feel too big/small or in the wrong order?
5. Should I expand S1–S2 next (command-level), since they underpin everything?

---

## 13. Sources (minikube docs used to verify §3)
- [minikube start flags](https://minikube.sigs.k8s.io/docs/commands/start/)
- [Static IP tutorial](https://minikube.sigs.k8s.io/docs/tutorials/static_ip/)
- [Network policy / CNI handbook](https://minikube.sigs.k8s.io/docs/handbook/network_policy/)
- [Docker driver](https://minikube.sigs.k8s.io/docs/drivers/docker/)
- [Configuration (`--extra-config`)](https://minikube.sigs.k8s.io/docs/handbook/config/)
