# DevOps

Fetch up to 6 submodules at a time with -j6

```shell
git clone --recurse-submodules -j6 https://github.com/nitinkc/DevOps.git
```

# Dockerhub
(https://hub.docker.com/repositories/nitinkc)[https://hub.docker.com/repositories/nitinkc]

# Curriculum Map

Submodule learning repos and the recommended study order. Each repo builds its own
MkDocs site (`mkdocs serve` inside the repo, see its `requirements.txt`).

## Core path (existing)

| Order | Repo | Covers |
|-------|------|--------|
| 1 | `DockerConcepts` | Docker fundamentals → images → storage → networking → compose |
| 2 | `DevOpsLearning` | DevOps theory spine + labs 00–11, interview prep, final quiz |
| 3 | `KubernetesConcepts` | Flagship lab site: Docker underlay → Minikube/Cilium → Istio → Flux → GKE handoff; manifest library; 13 troubleshooting scenarios |
| 4 | `HelmCharts` | Helm fundamentals → templating → Flux/GKE → operations → interview bank |
| 5 | `terraform-concepts` | TF basics → GCP networking → 5-root Acme multirepo (VPC/GKE/DNS/AR → secrets → CloudSQL → apps) |
| — | `go-k8s-server`, `k8s-frontend` | Toy apps used by pipeline/deploy labs |
| — | `jenkins-learning` | Spring Boot app; secondary/reference (GitLab CI is the primary CI track) |

## Expansion phases (implemented)

| Phase | Theme | Delivered |
|-------|-------|-----------|
| 1 | CI/CD foundations | `gitlab-spring-app-flux`: real pipeline (kaniko → security → helm → gated deploys + rollback) + 7-lesson docs site; `DevOpsLearning` theory 10 + labs 12–13; `jenkins-learning` Jenkinsfile + translation doc |
| 2 | Supply-chain security | `DockerConcepts` lesson 07 + 3 labs (distroless/trivy/cosign+SBOM); `gitlab-spring-app-flux` security stage (trivy gate, cosign sign, secret detection); `terraform-concepts` ref 12 (checkov/conftest/plan-artifact); `DevOpsLearning` theory 11–12 |
| 3 | Secrets management | `KubernetesConcepts` labs 12–13 (ESO, sops+age); `terraform-concepts` ref 13 (Vault dev lab) + acme backend ESO wiring; `DevOpsLearning` theory 13 |
| 4 | Delivery & reliability | `KubernetesConcepts` labs 14–17 (Flagger canary, PDBs/drains, Velero DR, RBAC/cert-manager/operators); `HelmCharts` testing-and-promotion + helm-unittest suite; `DevOpsLearning` theory 14 + lab 14 (SLO/incident) |
| 5 | Observability 2.0 & FinOps | `DevOpsLearning` labs 15–16 (OTel tracing, billing→BigQuery→Grafana + OpenCost); infracost in `terraform-concepts` ref 12 |
| 6 | Policy, promotion & capstone | `KubernetesConcepts` labs 18–19 (Kyverno, Kustomize multi-env promotion); `gitlab-spring-app-flux` capstone |


### deployment.yaml
```yaml
apiVersion: apps/v1  # Specifies the API version
kind: Deployment  # Defines the type of Kubernetes object
metadata:
  name: springboot-deployment  # Name of the deployment
spec:
  replicas: 2  # Number of pod replicas
  selector:
    matchLabels:
      app: springboot  # Matches pods with the label 'app: springboot'
  template:
    metadata:
      labels:
        app: springboot  # Labels the pods
    spec:
      containers:
      - name: springboot-container  # Name of the container
        image: nitinkc/k8s-helloworld:k8s-hw-v6  # Container image
        ports:
        - containerPort: 8080  # Container port
```

### service.yaml

A Service has exactly one `type`. Pick one per manifest:

```yaml
apiVersion: v1  # Specifies the API version
kind: Service  # Defines the type of Kubernetes object
metadata:
  name: springboot-service  # Name of the service
spec:
  selector:
    app: springboot  # Matches pods with the label 'app: springboot'
  ports:
    - protocol: TCP  # Specifies the protocol
      port: 80  # Port on which the service is exposed
      targetPort: 8080  # Port on the container to which traffic should be directed
  type: ClusterIP  # Service type — see variants below
```

Service type variants (choose exactly one):

```yaml
type: ClusterIP      # Default. Exposes the service on an internal cluster IP only.
type: NodePort       # Exposes the service on each node's IP at a static port (30000-32767).
type: LoadBalancer   # Exposes the service externally using a cloud provider's load balancer.
type: ExternalName   # Maps the service to a DNS name via the externalName field.
```

Headless Service (no cluster IP, used for direct pod DNS records, e.g. StatefulSets):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-headless
spec:
  selector:
    app: springboot
  clusterIP: None   # 'None' makes the service headless
  ports:
    - port: 80
      targetPort: 8080
```
