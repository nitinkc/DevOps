
The kube control command `kubectl` relies on a local configuration file on your Mac called the kubeconfig file, 
typically located at `~/.kube/config`

config file has a setting called the `current-context`. Whenever you type `kubectl get pods`, kubectl looks at your config file,
reads whatever name is set as the current-context, and sends your command to that specific cluster.

get the version (no `--` involved)
```shell
kubectl version 
```

When you create or connect to a cluster using Minikube, Kind, or GCP, they automatically write or update your `~/.kube/config`
file for you:

- **Minikube (`minikube start`):** Automatically creates a context named minikube and sets it as your active current-context.
- **Kind (`kind create cluster --name my-cluster`):** Automatically creates a context named kind-my-cluster and switches your terminal to it.
- **GCP / GKE (`gcloud container clusters get-credentials my-gke-cluster`):** Downloads the cloud credentials, adds a context for your GKE cluster, and automatically makes it your current-context.

See what cluster you are currently talking to:
```shell
kubectl config current-context
kubectl config get-contexts
```

Manually switch between clusters:
```shell
kubectl config use-context minikube
# or
kubectl config use-context kind-my-cluster
# or
kubectl config use-context gke_my-project_us-central1_my-cluster
```

## Common commands
```shell
kubectl get --help # See all possible combinations
kubectl get nodes
kubectl get nodes -w --watch=false
```

## Yaml files

4 important blocks
- apiVersion: 
- kind: Pod, ReplicaSet
- metadata:
- spec:


## NameSpace

Namespace : [Docker image -> Pod -> Deployment -> Service ]

```shell
kubectl get namespace

kubectl get pods -n default
```