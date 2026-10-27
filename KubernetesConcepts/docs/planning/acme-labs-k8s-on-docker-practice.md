# Acme Labs — Kubernetes-on-Docker Practice Lab (kind)

> **Note:** this file is written for **kind**. If you use **minikube**, follow `acme-labs-minikube-learning-plan.md` (stage order + a kind→minikube command translation table in its §3b); the lab content below stays valid.

Practise the **Kubernetes layer** of the GCP plan on your laptop first: cluster with chosen CIDRs, routes and "doors", NetworkPolicy, Istio mesh, 4-container pods with central logging, LoadBalancer ingress, and Flux ordering. Companion to `acme-labs-plan.md` (GCP phases P3–P9 map to labs K1–K10 here).

---

## 0. Your question: can Kubernetes create a VPC or handle subnet-to-subnet communication?

**Short answer: no — Kubernetes sits *on top of* the network; it does not create it.**

| Question | Answer |
|---|---|
| Can Kubernetes create a VPC / subnet / route / NAT? | **No, not natively.** Those are cloud (or Docker) objects. Terraform does it on GCP. |
| Can anything *inside* Kubernetes create them? | Only by calling the cloud API: **Config Connector** (GCP), **Crossplane**, or the **Terraform/OpenTofu controller (Flux tf-controller)** turn VPCs into Kubernetes CRDs — but GCP still creates the VPC. Not the learning path you want; you want hand-written Terraform. |
| Does Kubernetes know about VPC subnets? | **No.** It only knows nodes, a **pod CIDR**, and a **service CIDR**. Which subnet a node sits in is invisible to it. |
| How do pods in different "subnets" talk? | Kubernetes assumes a **flat pod network**: every pod can reach every pod. The **CNI** (Cilium on GKE Dataplane V2) delivers it; on GKE the pod range is a *secondary range* of the node subnet, so the VPC routes it natively. |
| Who controls what may talk to what? | Inside the cluster: **NetworkPolicy** (+ Istio). Between subnets/VPCs: **VPC routes + firewall rules** (cloud layer). Two layers, different owners. |
| What *does* Kubernetes ask the cloud for? | The **cloud controller manager** creates a load balancer when you make a `Service type=LoadBalancer`, GKE creates firewall rules/routes for the cluster, etc. |

**So on Docker we simulate the layer *below* Kubernetes ourselves** — and that is the point of this lab:

| GCP concept | Local stand-in in this lab |
|---|---|
| VPC `acme-main` | Docker network `acme-main` (`10.10.0.0/24`) |
| Second subnet `ops` | Docker network `acme-ops` (`10.10.1.0/28`) |
| Routes / Door 4 (hybrid) | A **router container** with `ip_forward` + `ip route add` on the nodes |
| VPC firewall | `iptables` rules on the router container |
| Default internet route (deleted in P8) | `ip route del default` inside the kind node containers |
| Doors 2 / 3 (/32 route, tag-gated default route) | Specific route / default route on selected nodes (+ a node label as the "tag") |
| Cloud NAT (SNAT, static IP) | Docker's MASQUERADE + Cilium masquerade (pod → node IP) |
| Node pool + tags | kind worker nodes + node labels |
| Pod / service secondary ranges | kind `podSubnet` / `serviceSubnet` |
| GKE Dataplane V2 | **Cilium** (same engine) |
| Internal load balancer | **MetalLB** handing out IPs from the Docker subnet |
| Cloud Logging central bucket | A **Fluent Bit aggregator** Deployment printing everything |
| Terraform → Flux handshake | Flux on kind (GitLab repo) |

### What you will *not* be able to practise locally (do it on GCP)
Workload Identity / WIF, Private Google Access + private DNS VIP, real Cloud NAT with static IPs, Cloud Logging API, GKE private control plane + authorized networks, Artifact Registry. Everything else in the Kubernetes layer transfers 1:1.

---

## 1. Prereqs and sizing

- Docker (Desktop or Engine), **8 GB RAM free** for Docker (cluster + Istio + Cilium). 1 control-plane + 2 workers.
- Tools: `kind`, `kubectl`, `helm`, `istioctl`, `flux`, `jq`, `dig`.
  ```bash
  # macOS
  brew install kind kubectl helm istioctl fluxcd/tap/flux jq
  # Linux: use each project's release binary; verify with the --version commands below
  kind --version; kubectl version --client; helm version; istioctl version --remote=false; flux --version
  ```
- **Docker Desktop (macOS/Windows) caveat:** your host cannot reach container IPs (`10.10.x.x`). Run **every network test from a container** (the `probe` container in K3) or via `kubectl exec`. On Linux the host can reach them directly.
- **Linux caveat:** if kind nodes crash with "too many open files": `sudo sysctl fs.inotify.max_user_watches=524288 fs.inotify.max_user_instances=512`.
- Work in a folder, e.g. `mkdir -p ~/acme-k8s-lab && cd ~/acme-k8s-lab`.

Order: **K1 → K2 → K3 → K4 → K5 → K6 → K7 → K8 → K9 → K10 → K11 (stretch)**. ≈ 8–12 hours total.

---

## K1 — Docker network = VPC; cluster with explicit CIDRs

**Goal:** a kind cluster living on *your* subnet, with pod/service ranges you chose, no default CNI (so we can install Cilium, like GKE Dataplane V2).
**Nuances:** pod/service CIDRs are fixed at creation; the node subnet must not overlap them; nodes are just containers with IPs on a bridge.

```bash
# the "VPC" and the "ops" subnet
docker network create --driver bridge --subnet 10.10.0.0/24 --gateway 10.10.0.1 acme-main
docker network create --driver bridge --subnet 10.10.1.0/28 --gateway 10.10.1.1 acme-ops
```
`kind-acme.yaml`
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: acme
networking:
  disableDefaultCNI: true          # Cilium instead of kindnet (kindnet != Dataplane V2)
  podSubnet: "10.20.0.0/20"        # same as the GCP gke-pods range
  serviceSubnet: "10.30.0.0/24"    # same as gke-svcs
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```
```bash
# attach the cluster to OUR docker network (experimental env var, supported by kind)
KIND_EXPERIMENTAL_DOCKER_NETWORK=acme-main kind create cluster --config kind-acme.yaml
kubectl config use-context kind-acme
kubectl get nodes            # NotReady is expected: no CNI yet

helm repo add cilium https://helm.cilium.io && helm repo update
helm install cilium cilium/cilium -n kube-system \
  --set ipam.mode=kubernetes \
  --set hubble.relay.enabled=true --set hubble.ui.enabled=true
kubectl -n kube-system rollout status ds/cilium --timeout=300s
kubectl get nodes            # Ready now

# "node pool" label (stand-in for GKE node pool / tags)
kubectl label node acme-worker  pool=main
kubectl label node acme-worker2 pool=main
```
Hints: if Cilium pods crash-loop, your Docker kernel may be too old for eBPF — fall back to Calico (`disableDefaultCNI` stays, install the Calico manifest with the same pod CIDR) and note what you lose (Hubble, identity-based policy).
**Verify:** `docker network inspect acme-main --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'` shows the 3 kind nodes in `10.10.0.0/24`.
**Gate G-K1:** explain why a second cluster on the same Docker network with pod CIDR `10.20.0.0/20` would be a problem (overlap) — and which GCP object enforces the same rule.

---

## K2 — Read the network you built

**Goal:** see the three address spaces (node / pod / service) and who owns each. (GCP: IPAM table in plan §2.)

```bash
kubectl get nodes -o wide                                     # INTERNAL-IP = 10.10.0.x
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD_CIDR:.spec.podCIDR,POOL:.metadata.labels.pool
kubectl get svc kubernetes -o jsonpath='{.spec.clusterIP}{"\n"}'     # in 10.30.0.0/24
kubectl get pods -A -o wide                                   # some pods show NODE IPs, others 10.20.x
docker exec acme-worker ip -br addr                           # eth0 = node IP on acme-main
docker exec acme-worker ip route                              # default via 10.10.0.1  <- the "internet gateway"
docker exec acme-worker ip route | grep 10.20                 # routes/devices Cilium added for pod CIDRs
```
Questions to answer in your own words:
1. Which pods have a node IP, and why? (hint: `hostNetwork: true`)
2. What is `/24` per node of the `/20` pod range? How many nodes max? (compare plan: `max_pods_per_node=64` → /25 on GKE)
3. Which of the three ranges is routable by anything outside the cluster? (none of pods/services — this is why masquerade exists, K4.)

**Gate G-K2:** you can fill an IPAM table (node, pod, service, router/proxy) for this lab from memory.

---

## K3 — Second subnet, a router, and a firewall (routes + firewall layer)

**Goal:** reproduce "subnet ↔ subnet", "route next hop" and "firewall rules" which Kubernetes itself knows nothing about. (GCP: P3 routes/firewall, Door 4.)

```
 acme-ops 10.10.1.0/28                         acme-main 10.10.0.0/24
 [probe 10.10.1.3] ── 10.10.1.2 [vpc-router] 10.10.0.250 ── [acme-control-plane][acme-worker][acme-worker2]
                                  ip_forward + iptables  (route next hop + firewall)
```
```bash
# router: dual-homed container with IP forwarding
docker run -d --name vpc-router --cap-add NET_ADMIN --sysctl net.ipv4.ip_forward=1 \
  --network acme-main --ip 10.10.0.250 nicolaka/netshoot sleep infinity
docker network connect --ip 10.10.1.2 acme-ops vpc-router

# a VM in the ops subnet (IAP-bastion stand-in)
docker run -d --name probe --cap-add NET_ADMIN --network acme-ops --ip 10.10.1.3 nicolaka/netshoot sleep infinity

# routes: "next hop = router" in both directions
docker exec probe ip route add 10.10.0.0/24 via 10.10.1.2
for n in $(kind get nodes --name acme); do docker exec $n ip route add 10.10.1.0/28 via 10.10.0.250; done

CP_IP=$(docker inspect -f '{{(index .NetworkSettings.Networks "acme-main").IPAddress}}' acme-control-plane)
docker exec probe ping -c2 $CP_IP
docker exec probe curl -sk -m5 -o /dev/null -w '%{http_code}\n' https://$CP_IP:6443/version   # any HTTP code = reachable; 000 = blocked
```
**Firewall exercise (iptables on the router = VPC firewall):**
```bash
docker exec vpc-router iptables -P FORWARD DROP                                  # "deny all" between subnets
docker exec vpc-router iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
docker exec vpc-router iptables -A FORWARD -p icmp -j ACCEPT
docker exec probe ping -c2 $CP_IP                                               # works
docker exec probe curl -sk -m5 -o /dev/null -w '%{http_code}\n' https://$CP_IP:6443/version   # 000: blocked
docker exec vpc-router iptables -A FORWARD -s 10.10.1.3 -d $CP_IP -p tcp --dport 6443 -j ACCEPT   # = "authorized network"
docker exec probe curl -sk -m5 -o /dev/null -w '%{http_code}\n' https://$CP_IP:6443/version   # reachable again
docker exec vpc-router iptables -L FORWARD -n -v --line-numbers
```
Hints:
- If `iptables` errors, try `iptables-legacy` (nft/legacy mismatch on some hosts).
- Routes live in the container's network namespace: **they are lost when a node container restarts.** Keep the `docker exec … ip route …` lines in `routes.sh` and re-run.
- Break it: remove the `ESTABLISHED,RELATED` rule → replies are dropped → "ping works one way" confusion = the exact symptom of an asymmetric firewall.
**Nuance revealed:** Kubernetes saw nothing of this. API-server reachability from a "management subnet" is decided by route + firewall, exactly as the GKE master authorized-networks + firewall story.
**Gate G-K3:** predict, before running, what `curl` returns after each firewall change.

---

## K4 — Egress: delete the default route, open the four doors

**Goal:** the plan's P8 in miniature. Cloud NAT ⇒ Docker's MASQUERADE; "tags" ⇒ node labels; "default route deleted" ⇒ `ip route del default`.

Preload everything the nodes will need **before** cutting egress (nodes can't pull images afterwards — a real lesson):
```bash
docker pull curlimages/curl && kind load docker-image curlimages/curl --name acme
docker pull nginxinc/nginx-unprivileged:alpine && kind load docker-image nginxinc/nginx-unprivileged:alpine --name acme
docker run -d --name echo --network acme-main --ip 10.10.0.201 nginx:alpine     # an "internal service" that logs client IPs
```
Helper (run a curl pod pinned to a given node):
```bash
probe() { n=$1; shift
  kubectl run p-$RANDOM --rm -i --restart=Never -q --image=curlimages/curl --image-pull-policy=Never \
    --overrides="{\"spec\":{\"nodeSelector\":{\"kubernetes.io/hostname\":\"$n\"}}}" -- -sS -m 5 -o /dev/null -w '%{http_code}\n' "$@" 2>&1 | tail -1; }
EX=$(dig +short example.com | head -1); echo $EX
```
| Step | Command | Expected |
|---|---|---|
| 0 baseline (training wheels) | `probe acme-worker https://example.com` | `200` (default route → Docker gateway → host NAT) |
| 1 who do I look like? | `probe acme-worker http://10.10.0.201` then `docker logs echo --tail 2` | client IP = **node IP**, not pod IP (SNAT). If you see a pod IP → `kubectl -n kube-system exec ds/cilium -- cilium config view \| grep -i masq` |
| 2 close the doors | `for n in acme-worker acme-worker2; do docker exec $n ip route del default; done` | |
| 3 | `probe acme-worker https://example.com` | `000`/timeout — likely **DNS still resolves** (Docker's resolver is outside the node route) but TCP has no route |
| 4 **Door 2** (/32) | `docker exec acme-worker ip route add $EX/32 via 10.10.0.1`; `probe acme-worker --resolve example.com:443:$EX https://example.com` | works for that IP only |
| 5 **Door 3** (tag-gated default) | `kubectl label node acme-worker2 allowed-open-nat=true`; `docker exec acme-worker2 ip route add default via 10.10.0.1`; `probe acme-worker2 https://example.com` vs `probe acme-worker …` | works on worker2 only |
| 6 **Door 4** (hybrid via router) | `probe acme-worker http://10.10.1.3` (the ops-subnet "on-prem" host; needs K3 routes) | reaches via the router route, unaffected by the missing default |
| 7 **Door 1** analogue | images arrive by `kind load` (side channel) — no Private-Google-Access equivalent locally | **Do this one on GCP (P3/P8)** |

Hints: if Cilium agents misbehave after the route is gone ("unable to determine direct routing device"), pin devices: `helm upgrade cilium cilium/cilium -n kube-system --reuse-values --set devices=eth0`.
**Break-it:** `kubectl delete pod` something that needs an image not preloaded → `ImagePullBackOff` — the same failure you will see on GCP when Door 1/AR mirror is missing.
**Optional (masquerade):** `helm upgrade cilium … --reuse-values --set enableIPv4Masquerade=false`, restart agents, repeat step 1 → replies to an un-routable pod IP are lost. That is why GKE needs `ip-masq-agent` for on-prem traffic (S2).
**Gate G-K4:** explain why a pod inherits its **node's** egress ability (it is the node's route table that decides), and why per-pod egress control needs NetworkPolicy/Istio, not routes or tags.

---

## K5 — Three domains: workloads on a flat network

**Goal:** the three namespaces, service accounts and Services; observe that Kubernetes is **flat by default** — everything can reach everything.

```bash
kubectl create ns observability   # used in K8
for d in orders payments inventory; do
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata: { name: dom-$d }
---
apiVersion: v1
kind: ServiceAccount
metadata: { name: $d, namespace: dom-$d }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: $d, namespace: dom-$d }
spec:
  replicas: 1
  selector: { matchLabels: { app: $d } }
  template:
    metadata: { labels: { app: $d, domain: $d } }
    spec:
      serviceAccountName: $d
      containers:
      - name: app
        image: nginxinc/nginx-unprivileged:alpine     # listens on 8080, has busybox wget
        imagePullPolicy: IfNotPresent
        ports: [{ containerPort: 8080 }]
        resources: { requests: { cpu: 50m, memory: 64Mi } }
---
apiVersion: v1
kind: Service
metadata: { name: $d, namespace: dom-$d }
spec:
  selector: { app: $d }
  ports: [{ name: http, port: 80, targetPort: 8080 }]
EOF
done
kubectl get pods -A -l 'domain'

# helper: call <from-ns> <from-deploy> <url>  -> prints the last line wget prints
call() { kubectl -n $1 exec deploy/$2 -c app -- sh -c "wget -O /dev/null -T3 $3 2>&1 | tail -n1"; }
call dom-orders   orders   http://payments.dom-payments/
call dom-orders   orders   http://inventory.dom-inventory/
call dom-payments payments http://inventory.dom-inventory/        # works too — flat network!
```
**Nuance revealed:** Service `port 80` → pod `8080`. Remember this for K6: **NetworkPolicy ports are *pod* ports (8080), not Service ports.**
**Gate G-K5:** list three reasons "it works" here is a *problem* for a three-domain platform.

---

## K6 — NetworkPolicy: default-deny + explicit allows (L3/L4 guard)

**Goal:** call graph `orders → payments`, `orders → inventory`; `payments ↔ inventory` denied. Cilium enforces it (as on GKE Dataplane V2).

```bash
for ns in dom-orders dom-payments dom-inventory; do
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: $ns }
spec: { podSelector: {}, policyTypes: [Ingress, Egress] }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-dns, namespace: $ns }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
    ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
EOF
done

# orders may call the other two domains on pod port 8080
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: orders-egress, namespace: dom-orders }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchExpressions:
        - { key: kubernetes.io/metadata.name, operator: In, values: [dom-payments, dom-inventory] }
    ports: [{ protocol: TCP, port: 8080 }]
EOF
for d in payments inventory; do
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-from-orders, namespace: dom-$d }
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - from: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: dom-orders } } }]
    ports: [{ protocol: TCP, port: 8080 }]
EOF
done
```
**Verify matrix** (predict first):
```bash
call dom-orders   orders   http://payments.dom-payments/       # OK
call dom-orders   orders   http://inventory.dom-inventory/     # OK
call dom-payments payments http://inventory.dom-inventory/     # timeout (egress default-deny)
call dom-inventory inventory http://payments.dom-payments/     # timeout
# see the drops (agent has the hubble CLI; or install hubble and port-forward)
kubectl -n kube-system exec ds/cilium -- hubble observe --verdict DROPPED --last 10
```
Break-it exercises:
1. Delete `allow-dns` in `dom-orders` → calls fail with "bad address" (DNS), not a timeout. Different symptom, same root cause class.
2. Change the policy port to `80` → blocked. (Service port vs pod port.)
**Gate G-K6:** you can tell, from the error text alone, DNS-denied vs connection-denied vs not-allowed-by-server.

---

## K7 — Istio sidecars: mTLS and identity-based authorization (L7 guard)

**Goal:** the same call graph protected by **who** is calling, not just **where** from. (GCP: P5/P7.)

```bash
istioctl install --set profile=minimal -y         # istiod only (sidecar mode on purpose)
kubectl -n istio-system get pods
kubectl label ns dom-orders dom-payments dom-inventory istio-injection=enabled
for ns in dom-orders dom-payments dom-inventory; do kubectl -n $ns rollout restart deploy; done
kubectl -n dom-orders get pods         # watch: pods stay 1/2 Ready ... why?
```
**Expected break #1:** default-deny **egress** blocks sidecar → istiod (15012), so Envoy never gets config. Prove it, then fix:
```bash
istioctl proxy-status
kubectl -n dom-orders logs deploy/orders -c istio-proxy --tail=5      # connection timeouts to istiod
for ns in dom-orders dom-payments dom-inventory; do
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-istiod, namespace: $ns }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: istio-system } } }]
    ports: [{ protocol: TCP, port: 15012 }]
EOF
done
kubectl -n dom-orders get pods         # 2/2
istioctl proxy-status                  # SYNCED
call dom-orders orders http://payments.dom-payments/               # still OK
```
**STRICT mTLS** (and the "plain client" test):
```bash
kubectl apply -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default, namespace: dom-payments }
spec: { mtls: { mode: STRICT } }
EOF
# temporary hole in the NetworkPolicy so the plain client can reach the port at all
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: tmp-allow-from-default, namespace: dom-payments }
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - from: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: default } } }]
    ports: [{ protocol: TCP, port: 8080 }]
EOF
kubectl run plain --rm -i --restart=Never -q --image=curlimages/curl --image-pull-policy=Never -- -sS -m4 http://payments.dom-payments/   # reset / empty reply = STRICT mTLS
kubectl -n dom-payments delete netpol tmp-allow-from-default
call dom-orders orders http://payments.dom-payments/               # meshed client: OK
```
**Identity-based AuthorizationPolicy** — an "intruder" in the *same namespace* as orders (the network layer cannot tell them apart; the identity can):
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata: { name: intruder, namespace: dom-orders }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: intruder, namespace: dom-orders }
spec:
  replicas: 1
  selector: { matchLabels: { app: intruder } }
  template:
    metadata: { labels: { app: intruder } }
    spec:
      serviceAccountName: intruder
      containers:
      - { name: app, image: nginxinc/nginx-unprivileged:alpine, imagePullPolicy: IfNotPresent }
EOF
kubectl apply -f - <<'EOF'
apiVersion: security.istio.io/v1        # older Istio: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: { name: only-orders, namespace: dom-payments }
spec:
  action: ALLOW
  rules:
  - from: [{ source: { principals: ["cluster.local/ns/dom-orders/sa/orders"] } }]
EOF
call dom-orders orders   http://payments.dom-payments/     # OK
call dom-orders intruder http://payments.dom-payments/     # "403 Forbidden" (RBAC: access denied) — network allowed, identity not
```
Hints: `istioctl analyze -A`; `istioctl x describe pod <pod> -n dom-orders`; `kubectl -n dom-payments logs deploy/payments -c istio-proxy`.
**Nuance revealed:** two guards, two failure signatures: **timeout** (NetworkPolicy) vs **403** (AuthorizationPolicy) vs **reset** (STRICT mTLS vs plaintext).
**Gate G-K7:** for each of the three failures, name the layer without running a command.

---

## K8 — Four-container pod and central logging

**Goal:** `app` + `istio-proxy` (injected) + `fluent-bit` + `otel-collector` in one pod; everything lands in **one central place** (stand-in for Cloud Logging bucket + sink).

### 8a. Central aggregator (the "log bucket")
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata: { name: aggregator-config, namespace: observability }
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush 1
    [INPUT]
        Name   forward
        Listen 0.0.0.0
        Port   24224
    [OUTPUT]
        Name   stdout
        Match  *
        Format json_lines
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: log-aggregator, namespace: observability }
spec:
  replicas: 1
  selector: { matchLabels: { app: log-aggregator } }
  template:
    metadata: { labels: { app: log-aggregator } }
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:3.2
        ports: [{ containerPort: 24224 }]
        volumeMounts: [{ name: cfg, mountPath: /fluent-bit/etc/fluent-bit.conf, subPath: fluent-bit.conf }]
      volumes: [{ name: cfg, configMap: { name: aggregator-config } }]
---
apiVersion: v1
kind: Service
metadata: { name: log-aggregator, namespace: observability }
spec:
  selector: { app: log-aggregator }
  ports: [{ name: tcp-forward, port: 24224, targetPort: 24224 }]
EOF
```
### 8b. The 4-container pod in `dom-orders`
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata: { name: logdemo-fluentbit, namespace: dom-orders }
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush 1
        Parsers_File parsers.conf
    [INPUT]
        Name   tail
        Path   /var/log/app/*.log
        Parser json
        Tag    app.${DOMAIN}
    [FILTER]
        Name   record_modifier
        Match  *
        Record domain ${DOMAIN}
    [OUTPUT]
        Name   forward
        Match  *
        Host   log-aggregator.observability.svc.cluster.local
        Port   24224
---
apiVersion: v1
kind: ConfigMap
metadata: { name: logdemo-otel, namespace: dom-orders }
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          http: { endpoint: 127.0.0.1:4318 }
    exporters:
      debug: { verbosity: basic }
    service:
      pipelines:
        traces: { receivers: [otlp], exporters: [debug] }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: logdemo, namespace: dom-orders }
spec:
  replicas: 1
  selector: { matchLabels: { app: logdemo } }
  template:
    metadata: { labels: { app: logdemo } }
    spec:
      serviceAccountName: orders
      volumes:
      - { name: applogs, emptyDir: {} }
      - { name: fb-cfg,  configMap: { name: logdemo-fluentbit } }
      - { name: otel-cfg, configMap: { name: logdemo-otel } }
      containers:
      - name: app                                   # writes JSON to stdout AND a file, emits one OTLP span
        image: curlimages/curl
        imagePullPolicy: Never
        env: [{ name: DOMAIN, value: orders }]
        command: ["/bin/sh", "-c"]
        args:
        - |
          while true; do
            echo "{\"severity\":\"INFO\",\"message\":\"tick\",\"domain\":\"${DOMAIN}\"}" | tee -a /var/log/app/app.log
            curl -s -m2 -H 'Content-Type: application/json' http://127.0.0.1:4318/v1/traces -d '{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"orders"}}]},"scopeSpans":[{"spans":[{"traceId":"5B8EFFF798038103D269B633813FC60C","spanId":"EEE19B7EC3C1B174","name":"tick","startTimeUnixNano":"1700000000000000000","endTimeUnixNano":"1700000001000000000","kind":1}]}]}]}' >/dev/null
            sleep 5
          done
        volumeMounts: [{ name: applogs, mountPath: /var/log/app }]
      - name: fluent-bit                            # file-tail sidecar
        image: fluent/fluent-bit:3.2
        env: [{ name: DOMAIN, value: orders }]
        volumeMounts:
        - { name: applogs, mountPath: /var/log/app, readOnly: true }
        - { name: fb-cfg,  mountPath: /fluent-bit/etc/fluent-bit.conf, subPath: fluent-bit.conf }   # subPath keeps parsers.conf
      - name: otel-collector                        # OTLP sidecar
        image: otel/opentelemetry-collector:latest   # pin a version once it works
        args: ["--config=/conf/config.yaml"]
        volumeMounts: [{ name: otel-cfg, mountPath: /conf }]
EOF
kubectl -n dom-orders get pod -l app=logdemo -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'   # app fluent-bit otel-collector istio-proxy
```
**Expected break #2:** the fluent-bit sidecar cannot reach the aggregator (default-deny egress from `dom-orders`). Fix by allowing it, then watch the central log:
```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-to-observability, namespace: dom-orders }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: observability } } }]
    ports: [{ protocol: TCP, port: 24224 }]
EOF
kubectl -n dom-orders logs deploy/logdemo -c otel-collector --tail=5        # "Traces" lines = OTLP path alive
kubectl -n observability logs deploy/log-aggregator --tail=5                # JSON lines with "domain":"orders" = file-tail path alive
kubectl -n dom-orders logs deploy/logdemo -c app --tail=3                   # stdout path (what the GKE node agent would ship)
```
Exercises:
- Turn this pod spec into the **Helm chart** `acme-microservice` (plan P7a): `helm create acme-microservice`, delete the defaults, keep `values.yaml` toggles `sidecars.otel.enabled`, `sidecars.fluentbit.enabled`, `domain`, `image.*`; verify with `helm lint` and `helm template . --set domain=payments | kubectl apply --dry-run=client -f -`.
- Add the same pod to `dom-payments` with `domain=payments` and filter by the `domain` field at the aggregator.
- Real images: build your own Go/Python/Node service and `docker build` → `kind load docker-image` (the local stand-in for Artifact Registry).
**Gate G-K8:** explain the three log paths (stdout, file-tail, OTLP) and which of them needs a NetworkPolicy hole.

---

## K9 — LoadBalancer ingress: MetalLB (internal LB analogue)

**Goal:** the P9 "internal load balancer" and its three classic breaks, using the `ops` subnet as the client side.

```bash
# check the current release at github.com/metallb/metallb before pinning
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl -n metallb-system rollout status deploy/controller --timeout=180s
kubectl apply -f - <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: acme-pool, namespace: metallb-system }
spec: { addresses: ["10.10.0.240-10.10.0.249"] }
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: { name: acme-l2, namespace: metallb-system }
EOF
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata: { name: payments-lb, namespace: dom-payments }
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local          # keep the client's source IP
  selector: { app: payments }
  ports: [{ name: http, port: 80, targetPort: 8080 }]
EOF
kubectl -n dom-payments get svc payments-lb       # EXTERNAL-IP from 10.10.0.240-249
LB=$(kubectl -n dom-payments get svc payments-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
docker exec probe curl -s -m4 -o /dev/null -w '%{http_code}\n' http://$LB/
```
Expected breaks (diagnose each by symptom, then fix):

| # | Symptom from `probe` | Layer | Fix |
|---|---|---|---|
| 1 | timeout | **router firewall** (K3 `FORWARD DROP`) | `iptables -A FORWARD -s 10.10.1.3 -d $LB -p tcp --dport 80 -j ACCEPT` (+ established rule) |
| 2 | timeout (router now open) | **NetworkPolicy** ingress deny on `dom-payments` | add ingress from `ipBlock 10.10.1.0/28` on pod port 8080 |
| 3 | connection reset | **STRICT mTLS** vs plaintext client (an LB speaks plaintext) | workload-level `PeerAuthentication` with `portLevelMtls: { 8080: { mode: PERMISSIVE } }` (needs `selector.matchLabels app: payments`) |
| 4 | `403` | **AuthorizationPolicy** — plaintext request has no principal | add a second ALLOW rule `from: source: ipBlocks: ["10.10.1.0/28"]` (verify the source IP Envoy sees) |

**Nuance revealed:** every layer you built in K3–K7 appears again in the ingress path — firewall, NetworkPolicy, mTLS mode, authorization — which is why P9 on GCP also has 3–4 breaks.
**Gate G-K9:** from the symptom alone, name the failing layer for each row.

---

## K10 — Flux: let Git own the cluster (ordering lab)

**Goal:** replace every manual `kubectl apply` of K5–K9 with a Flux repo whose **ordering** is explicit. (GCP: P5.)
Note: the **CNI (Cilium) is installed outside Flux** — Flux pods themselves need a network. On GKE the equivalent is Dataplane V2 being a cluster setting.

```bash
export GITLAB_TOKEN=<PAT with api scope>
flux check --pre
flux bootstrap gitlab --hostname=gitlab.com \
  --owner=acme-labs-<handle>/domain-shared/configurations --repository=flux-releases-kind \
  --branch=main --path=clusters/kind --token-auth       # add --personal if the owner is a user, not a group
flux get kustomizations -A
```
Repo layout:
```
clusters/kind/
  flux-system/                    (generated)
  10-infra-controllers.yaml
  20-infra-configs.yaml
  30-apps.yaml
infrastructure/controllers/   sources.yaml  istio-base.yaml  istiod.yaml  metallb.yaml
infrastructure/configs/       namespaces.yaml  netpol.yaml  mesh-policy.yaml  metallb-pool.yaml
apps/kind/                    observability.yaml  orders.yaml  payments.yaml  inventory.yaml
```
`clusters/kind/10-infra-controllers.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: { name: infra-controllers, namespace: flux-system }
spec:
  interval: 10m
  path: ./infrastructure/controllers
  prune: true
  wait: true
  timeout: 10m
  sourceRef: { kind: GitRepository, name: flux-system }
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: { name: infra-configs, namespace: flux-system }
spec:
  interval: 10m
  dependsOn: [{ name: infra-controllers }]
  path: ./infrastructure/configs
  prune: true
  wait: true
  sourceRef: { kind: GitRepository, name: flux-system }
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: { name: apps, namespace: flux-system }
spec:
  interval: 10m
  dependsOn: [{ name: infra-configs }]
  path: ./apps/kind
  prune: true
  sourceRef: { kind: GitRepository, name: flux-system }
```
`infrastructure/controllers/sources.yaml` + Istio releases (MetalLB analogous, chart repo `https://metallb.github.io/metallb`, chart `metallb`)
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata: { name: istio, namespace: flux-system }
spec: { interval: 1h, url: https://istio-release.storage.googleapis.com/charts }
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata: { name: istio-base, namespace: flux-system }
spec:
  interval: 10m
  targetNamespace: istio-system
  install: { createNamespace: true }
  chart: { spec: { chart: base, sourceRef: { kind: HelmRepository, name: istio, namespace: flux-system } } }
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata: { name: istiod, namespace: flux-system }
spec:
  interval: 10m
  targetNamespace: istio-system
  dependsOn: [{ name: istio-base }]
  chart: { spec: { chart: istiod, sourceRef: { kind: HelmRepository, name: istio, namespace: flux-system } } }
  values:
    meshConfig: { accessLogFile: /dev/stdout, accessLogEncoding: JSON }
```
Migration exercise (do it in this order):
1. Commit the K5 namespaces (with `istio-injection: enabled` label), K6/K7 policies and K9 pool into `configs/`, the Deployments into `apps/kind/`; commit the controllers into `controllers/`.
2. `kubectl delete` the manually-applied objects (or recreate the cluster from K1) and watch: `flux get ks -A -w`, `flux tree ks apps`, `flux events`.
3. After each change: `flux reconcile ks apps --with-source`.

**Break-it exercises (the point of this lab):**
- Remove `dependsOn` from `infra-configs` → the pool fails with *"no matches for kind IPAddressPool"* (CRDs not yet installed). Put it back.
- Create the namespaces **after** the Deployments (move the label out) → pods start **without a sidecar**; restart them to fix. This is the plan's G5.
- Remove `wait: true` from controllers → dependants start before istiod is ready.
**Gate G-K10:** you can state, for each object, why it is in controllers vs configs vs apps.

---

## K11 — Stretch (local versions of the GCP stretch menu)

| Idea | How locally | GCP twin |
|---|---|---|
| **Two clusters / two VPCs** | second kind cluster on `acme-ops` network with different pod/service CIDRs, router between them | S2 hybrid, overlapping-CIDR pain |
| **Per-domain node pool** | label workers `pool=orders` etc., `nodeSelector` + taints per domain | S3 |
| **Egress with a fixed IP** | Cilium Egress Gateway (check current docs for status/requirements) | Cloud NAT static IPs |
| **Registry mirror** | `registry:2` container on `acme-main` + containerd mirror config (see kind "local registry" docs) | Artifact Registry + Door 1 |
| **Mesh REGISTRY_ONLY** | `meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY` + `ServiceEntry` | S5 |
| **Observability** | replace the aggregator with Loki + Grafana | Log Analytics |

---

## Gates summary (predict first, then verify)

| Gate | You can… |
|---|---|
| G-K1 | explain why CIDRs must not overlap (node / pod / service / peer) |
| G-K2 | fill the IPAM table from memory |
| G-K3 | predict each `curl` result across firewall changes |
| G-K4 | explain why pods inherit their node's egress and why tags can't do per-pod control |
| G-K5 | list three risks of a flat network |
| G-K6 | tell DNS-denied / connection-denied / server-denied apart from text |
| G-K7 | name the layer for timeout / reset / 403 |
| G-K8 | explain stdout vs file-tail vs OTLP |
| G-K9 | name the failing layer for each ingress symptom |
| G-K10 | justify controllers → configs → apps order |

When all ten pass, repeat on GCP (plan P3–P9). Expect only the cloud-specific pieces (WIF, Private Google Access, Cloud NAT, Cloud Logging, authorised networks) to be new.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Nodes `NotReady` after create | CNI not installed yet (K1) |
| Cilium pods CrashLoop | old kernel / Docker VM without eBPF → Calico fallback |
| Pods `ImagePullBackOff` after K4 | default route removed; image not `kind load`-ed |
| Routes vanished | node container restarted → re-run your `routes.sh` |
| Can't reach `10.10.x.x` from host | Docker Desktop VM → test from `probe`/`kubectl exec` |
| Sidecar missing | namespace label added after pods started → `rollout restart` |
| Pods 1/2 Ready after injection | egress to istiod:15012 blocked (K7 break #1) |
| `flux bootstrap` fails | token scope, group path, or `--personal` mismatch |

## Teardown
```bash
flux uninstall --silent 2>/dev/null          # optional
kind delete cluster --name acme
docker rm -f vpc-router probe echo
docker network rm acme-ops acme-main
```
