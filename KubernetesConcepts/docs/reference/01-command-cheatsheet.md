# 01 · The Complete CLI Command Cheat Sheet

> **Keep This Handy:** Every operational command used across Labs 01–10 and GCP GKE, grouped by architectural layer.

---

## 1. Docker Underlay, Subnets, Routing & Firewall (Lab 01 & 04)

```bash
# --- Create isolated Docker bridge networks ("Subnets") ---
docker network create --driver bridge --subnet 10.10.0.0/24 --gateway 10.10.0.1 acme-main
docker network create --driver bridge --subnet 10.10.1.0/28 --gateway 10.10.1.1 acme-ops

# --- Inspect attached containers and their IPv4 addresses ---
docker network inspect acme-main --format '{{range .Containers}}{{.Name}}\t{{.IPv4Address}}{{"\n"}}{{end}}'
docker network inspect acme-ops  --format '{{range .Containers}}{{.Name}}\t{{.IPv4Address}}{{"\n"}}{{end}}'

# --- Launch dual-homed router & ops probe bastion ---
docker run -d --name vpc-router --cap-add NET_ADMIN --sysctl net.ipv4.ip_forward=1 \
  --network acme-main --ip 10.10.0.250 nicolaka/netshoot sleep infinity
docker network connect --ip 10.10.1.2 acme-ops vpc-router

docker run -d --name probe --cap-add NET_ADMIN \
  --network acme-ops --ip 10.10.1.3 nicolaka/netshoot sleep infinity

# --- Inspect & manipulate Linux routing tables inside containers/nodes ---
docker exec probe ip -br addr
docker exec probe ip route
docker exec probe ip route get 10.10.0.10
docker exec probe ip route add 10.10.0.0/24 via 10.10.1.2
docker exec acme  ip route add 10.10.1.0/28 via 10.10.0.250

# --- Egress Lockdown: Delete & restore default internet gateway route ---
docker exec acme-m02 ip route del default
docker exec acme-m02 ip route add 93.184.215.14/32 via 10.10.0.1     # Door 2: /32 VIP
docker exec acme-m03 ip route add default via 10.10.0.1              # Door 3: Tagged Node NAT

# --- Stateful Firewall (iptables on vpc-router) ---
docker exec vpc-router iptables -P FORWARD DROP
docker exec vpc-router iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
docker exec vpc-router iptables -A FORWARD -p icmp -j ACCEPT
docker exec vpc-router iptables -A FORWARD -s 10.10.1.3 -d 10.10.0.10 -p tcp --dport 8443 -j ACCEPT
docker exec vpc-router iptables -L FORWARD -n -v --line-numbers
docker exec vpc-router iptables -D FORWARD <line-number>             # Delete specific rule

# --- Live Packet Capture (tcpdump) ---
docker exec vpc-router tcpdump -nn -i any icmp or port 80 or port 8443 -c 20
```

---

## 2. Minikube Lifecycle, Profiles & Node Management (Lab 02)

```bash
# --- Start 3-node cluster on custom subnet with explicit Pod & Service CIDRs ---
minikube start -p acme \
  --driver=docker \
  --container-runtime=containerd \
  --nodes=3 \
  --cpus=2 --memory=3072 \
  --network=acme-main \
  --static-ip=10.10.0.10 \
  --service-cluster-ip-range=10.30.0.0/24 \
  --extra-config=kubeadm.pod-network-cidr=10.20.0.0/20 \
  --extra-config=controller-manager.allocate-node-cidrs=true \
  --cni=bridge

# --- Profile & Node Inspection ---
minikube profile list
minikube status -p acme
minikube node list -p acme
minikube ssh -p acme -n acme-m02 -- ip route

# --- Load Locally Built Docker Images into All Minikube Nodes (Door 1 Local Twin) ---
minikube image load -p acme <image-name>:<tag>
minikube image ls -p acme

# --- Node Pool Labeling (Stand-in for GKE Node Pools & Network Tags) ---
kubectl label node acme-m02 pool=main
kubectl label node acme-m03 pool=main allowed-open-nat=true

# --- Stop / Start / Teardown ---
minikube stop -p acme          # Note: ephemeral container `ip route` additions reset on stop/start!
minikube delete -p acme
```

---

## 3. IPAM, Services, CoreDNS & Endpoint Triage (Lab 02 & 03)

```bash
# --- Verify the 3 Address Spaces (Nodes, Pods, Services) ---
kubectl get nodes -o custom-columns=NAME:.metadata.name,INTERNAL_IP:.status.addresses[0].address,POD_CIDR:.spec.podCIDR,POOL:.metadata.labels.pool,NAT_TAG:.metadata.labels.allowed-open-nat
kubectl get pods -A -o wide
kubectl get svc -A

# --- Service & EndpointSlice Verification (Catch Selector or TargetPort Mismatches) ---
kubectl get svc,endpoints,endpointslices -n dom-orders
kubectl describe svc orders -n dom-orders

# --- CoreDNS & ndots Resolution Debugging ---
kubectl run dns-test --rm -i --restart=Never --image=nicolaka/netshoot -- \
  dig +search +showsearch payments.dom-payments
kubectl run dns-test --rm -i --restart=Never --image=nicolaka/netshoot -- \
  cat /etc/resolv.conf

# --- Fast One-Shot HTTP Probe Pinned to a Specific Node ---
probe_node() {
  local node=$1; shift
  kubectl run p-$RANDOM --rm -i --restart=Never -q --image=curlimages/curl --image-pull-policy=IfNotPresent \
    --overrides="{\"spec\":{\"nodeSelector\":{\"kubernetes.io/hostname\":\"$node\"}}}" \
    -- -sS -m 5 -o /dev/null -w '%{http_code}\n' "$@" 2>&1 | tail -1
}

# --- Fast East-West Service Call from Inside an App Container ---
call_svc() {
  local ns=$1 deploy=$2 url=$3
  kubectl -n "$ns" exec "deploy/$deploy" -c app -- sh -c "wget -O /dev/null -T 3 $url 2>&1 | tail -n1"
}
```

---

## 4. Cilium CNI, eBPF Datapath & Hubble L3/L4 Observability (Lab 02 & 05)

```bash
# --- Install / Upgrade Cilium via Helm (Matches GKE Dataplane V2) ---
helm repo add cilium https://helm.cilium.io && helm repo update
helm upgrade --install cilium cilium/cilium -n kube-system \
  --set ipam.mode=kubernetes \
  --set ipv4NativeRoutingCIDR=10.20.0.0/20 \
  --set devices=eth0 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# --- Cilium Health, Config & eBPF Endpoint Map ---
kubectl -n kube-system rollout status ds/cilium --timeout=300s
kubectl -n kube-system exec ds/cilium -- cilium status --brief
kubectl -n kube-system exec ds/cilium -- cilium config view | grep -iE "masq|routing|ipam"
kubectl -n kube-system exec ds/cilium -- cilium endpoint list

# --- Hubble Real-Time Packet Verdict Inspection (THE Best NetworkPolicy Debugger) ---
kubectl -n kube-system exec ds/cilium -- hubble observe --verdict DROPPED --last 20
kubectl -n kube-system exec ds/cilium -- hubble observe --namespace dom-payments -f
kubectl -n kube-system exec ds/cilium -- hubble observe --from-namespace dom-orders --to-namespace dom-payments

# --- Inspect NetworkPolicies ---
kubectl get netpol -A
kubectl describe netpol -n dom-payments
```

---

## 5. Istio Service Mesh, mTLS & SPIFFE Authorization (Lab 06)

```bash
# --- Install Istio Sidecar Mesh (Minimal Control Plane Profile) ---
istioctl install --set profile=minimal \
  --set meshConfig.accessLogFile=/dev/stdout \
  --set meshConfig.accessLogEncoding=JSON -y

# --- Enable Automatic Sidecar Injection & Trigger Pod Recreate ---
kubectl label ns dom-orders dom-payments dom-inventory istio-injection=enabled --overwrite
for ns in dom-orders dom-payments dom-inventory; do kubectl -n $ns rollout restart deploy; done

# --- Verify Envoy Sidecar Sync & Mesh Configuration ---
istioctl proxy-status
istioctl analyze -A
POD=$(kubectl -n dom-orders get pod -l app=orders -o jsonpath='{.items[0].metadata.name}')
istioctl x describe pod "$POD" -n dom-orders

# --- Inspect Envoy Certificates (SPIFFE ID) & Live Access Logs ---
istioctl proxy-config secret "$POD.dom-orders"
kubectl -n dom-payments logs deploy/payments -c istio-proxy --tail=20

# --- List Mesh Security Policies ---
kubectl get peerauthentication,authorizationpolicy -A
```

---

## 6. Sidecars, Helm Packaging & Central Logging (Lab 03 & 07)

```bash
# --- Verify All 4 Containers Running in a Single Pod ---
kubectl -n dom-orders get pod -l app=orders -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'

# --- Tail Each Telemetry Path Independently ---
kubectl -n dom-orders logs deploy/orders -c app --tail=10               # Pattern A: App stdout
kubectl -n dom-orders logs deploy/orders -c istio-proxy --tail=10       # Pattern A: Envoy JSON access log
kubectl -n dom-orders logs deploy/orders -c fluent-bit --tail=10        # Pattern B: File-tail shipper status
kubectl -n dom-orders logs deploy/orders -c otel-collector --tail=15    # Pattern C: OTLP trace spans
kubectl -n observability logs deploy/log-aggregator --tail=20           # Central Log Bucket equivalent

# --- Helm Chart Authoring, Linting & Dry-Run Rendering ---
helm lint charts/acme-microservice
helm template orders charts/acme-microservice --namespace dom-orders -f charts/acme-microservice/values.yaml
helm upgrade --install orders charts/acme-microservice -n dom-orders --create-namespace
helm list -A
```

---

## 7. MetalLB LoadBalancer & North-South Ingress (Lab 08)

```bash
# --- Install MetalLB & Check Speaker/Controller Readiness ---
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl -n metallb-system wait --for=condition=ready pod --all --timeout=180s

# --- Inspect Assigned External VIP & Test from Ops Subnet ---
kubectl get ipaddresspools,l2advertisements -n metallb-system
kubectl get svc -A -o wide | grep LoadBalancer
LB_IP=$(kubectl -n dom-payments get svc payments-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
docker exec probe curl -i -sS -m 4 "http://$LB_IP/"
```

---

## 8. FluxCD GitOps & Reconciliation (Lab 09)

```bash
# --- Pre-Flight & Bootstrap ---
flux check --pre
flux bootstrap gitlab \
  --hostname=gitlab.com \
  --owner="acme-labs-<handle>/domain-shared/configurations" \
  --repository=flux-releases-local \
  --branch=main \
  --path=clusters/local \
  --token-auth

# --- Day-2 Operations: Status, Dependency Tree, Force Reconcile, Suspend/Resume ---
flux get all -A
flux get kustomizations -w
flux tree kustomization apps
flux events
flux reconcile source git flux-system
flux reconcile kustomization infra-controllers --with-source
flux reconcile kustomization infra-configs
flux reconcile kustomization apps
flux suspend kustomization apps
flux resume kustomization apps
```

---

## 9. Quick Equivalent Commands on GCP GKE (Phase P3–P9 Reference)

```bash
# --- Authenticate & Fetch GKE Cluster Credentials ---
gcloud auth login && gcloud auth application-default login
gcloud container clusters get-credentials acme-np --zone us-central1-a --project my-devops-journey-502420

# --- Inspect GCP Routes, Cloud NAT & Firewall Rules (Replaces `docker exec vpc-router ...`) ---
gcloud compute routes list --filter="network:acme-main" --format="table(name,destRange,priority,nextHopGateway.basename(),tags.list())"
gcloud compute routers get-nat-mapping-info acme-router --region us-central1
gcloud compute firewall-rules list --filter="network:acme-main" --format="table(name,direction,sourceRanges.list(),targetTags.list(),allowed[].map().firewall_rule().list())"

# --- Query Central Cloud Logging Bucket (Replaces `kubectl -n observability logs deploy/log-aggregator`) ---
gcloud logging read 'resource.type="k8s_container" AND resource.labels.namespace_name=~"^dom-"' --limit 10 --format json
```

---

## 10. Pod Lifecycle & Secret Debugging (Lab 03 drills — the FDE daily driver)

```bash
# --- The universal pod debug flow: state -> events -> app logs ---
kubectl get pods -n dom-orders
kubectl describe pod <pod> -n dom-orders | tail -20            # events: pull, schedule, mounts, probes
kubectl logs <pod> -c app -n dom-orders --previous             # logs of the LAST crashed container
kubectl get events -n dom-orders --sort-by=.lastTimestamp

# --- Lifecycle failure signatures ---
kubectl describe pod <pod> | grep -A5 "Last State"             # OOMKilled / Error + exit code
kubectl get endpoints <svc> -n <ns>                            # empty when readiness probe fails (pod up, svc dead)

# --- Secrets (local .env analog) ---
kubectl -n dom-orders create secret generic app-secrets --from-literal=DB_PASSWORD='s3cr3t'
kubectl -n dom-orders get secret app-secrets -o yaml           # base64 only — NOT encryption
kubectl -n dom-orders get secret app-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# --- GCP twins of the same triage ---
gcloud compute ssh probe --zone us-central1-a --tunnel-through-iap      # bastion access (probe twin)
gcloud logging read 'severity>=ERROR' --limit 20                       # Cloud Logging filter (aggregator twin)
gcloud network-management connectivity-tests create <name> --source-instance=probe \
  --destination-port=443 --destination-ip=10.10.0.240 --protocol TCP   # reachability simulation (tcpdump/Hubble twin)
```
