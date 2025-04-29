# DevOps

Fetch up to 6 submodules at a time with -j6

```shell
git clone --recurse-submodules -j6 https://github.com/nitinkc/DevOps.git
```

# Dockerhub
(https://hub.docker.com/repositories/nitinkc)[https://hub.docker.com/repositories/nitinkc]


deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
    spec:
      containers:
      - name: springboot-container
        image: nitinkc/k8s-helloworld:k8s-hw-v6
        ports:
        - containerPort: 8080

```

Servce YML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  selector:
    app: springboot
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: NodePort

```
