
![nodePort.jpeg](../nodePort.jpeg)

No Ingress needed

**No.** Frontend-to-backend communication happens entirely **inside** the cluster using Kubernetes internal DNS.

* **Internal Traffic (Frontend to Backend):** Uses cluster DNS (`http://go-backend-service:80`). You **do not** need Ingress for this.
* **External Traffic (You in your browser to the Frontend):** Uses `NodePort` (`http://localhost:30081`) or an Ingress controller if you want a clean domain name (like `myapp.local`) instead of a node port.
![nodePortCode.png](../nodePortCode.png)

## Deploy backend 

```shell
kubectl apply -f backend-deployment.yml

# Expose via thr service
kubectl port-forward svc/go-backend-service 30080:80
```

**The Traffic Flow:**

`External Request (localhost:30080) -> NodePort (30080) -> Service Port (80) -> Target Container Port (8080)`

## Port forwarding

### 1. Forwarding to the Service (svc/go-backend-service)

```shell
kubectl port-forward svc/go-backend-service 30080:80
```

### 2. Forwarding to a Pod directly (pod/go-server-xyz)
```shell
kubectl port-forward pod/go-server-abc123 30080:8080
```

```mermaid
graph TD
    subgraph Local Laptop ["Your Local Machine / Laptop"]
        User["Terminal / Browser<br/>`http://localhost:30080`"]
    end

    subgraph Minikube Cluster ["Kubernetes Cluster Boundary"]
        subgraph ServiceLayer ["Service: go-backend-service (Type: NodePort)"]
            NodePort["NodePort: 30080<br/>*(Cluster Node Edge)*"]
            SvcPort["Service Port: 80<br/>*(Internal Cluster IP)*"]
        end

        subgraph DeploymentLayer ["Deployment: go-server-deployment (Replicas: 3)"]
            Pod1["Pod 1<br/>Container Port: 8080"]
            Pod2["Pod 2<br/>Container Port: 8080"]
            Pod3["Pod 3<br/>Container Port: 8080"]
        end
    end

    %% Connections & Flow
    User -- "kubectl port-forward<br/>`30080:80`" --> SvcPort
    NodePort -.->|Alternative Direct Route| SvcPort
    SvcPort -->|Load Balances Traffic| Pod1
    SvcPort -->|Load Balances Traffic| Pod2
    SvcPort -->|Load Balances Traffic| Pod3
```