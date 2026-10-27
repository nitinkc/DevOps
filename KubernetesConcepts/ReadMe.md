

## Install minikube

```shell
brew install minikube

minikube start
minikube dashboard

minikube dashboard --url
minikube dashboard --port=8080

minikube addons enable metrics-server


minikube addons enable yakd
minikube service yakd-dashboard -n yakd-dashboard

```

## Clean Reset
```shell
# Completely delete the existing cluster and its configuration cache
minikube delete --all --purge

# Start a completely fresh cluster
minikube start --driver=docker

```