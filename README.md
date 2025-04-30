# DevOps

Fetch up to 6 submodules at a time with -j6

```shell
git clone --recurse-submodules -j6 https://github.com/nitinkc/DevOps.git
```

# Dockerhub
(https://hub.docker.com/repositories/nitinkc)[https://hub.docker.com/repositories/nitinkc]


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
  type: NodePort  # Specifies the service type (NodePort)
  type: ClusterIP  # Default service type. Exposes the service on an internal IP in the cluster.
  type: NodePort  # Exposes the service on each node's IP at a static port.
  type: LoadBalancer  # Exposes the service externally using a cloud provider's load balancer.
  type: ExternalName  # Maps the service to the contents of the externalName field (e.g., DNS name).
  # Headless Service
  type: ClusterIP  # Set to 'None' to create a headless service.
  clusterIP: None  # Creates a headless service, which does not use a ClusterIP.
```
