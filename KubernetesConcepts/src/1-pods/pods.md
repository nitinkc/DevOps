

Check logs of a container inside a pod
```shell
kubectl logs <pod_name>

kubectl logs <pod_name> -c <container_name>

# Real time logs
kubectl logs -f <pod_name>
kubectl logs <pod_name> --tail=50
kubectl logs <pod_name> --timestamps

# Logs of crashed container
kubectl logs <pod_name> -p

```


Check containers inside a pod
```shell
kubectl describe pod <pod_name>

kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'
```

## Status

CrashLoopBackOff

ErrImagePull


## Port Forwarding

```shell
kubectl port-forward pod/pod-go-server-test 8080:8080
```

## Test end points

```shell
curl http://localhost:8080/
```

```shell
curl -i http://localhost:8080/health
```

## Pod
Pod has a one-to-one relationship with a node
- a running pod cannot be moved from one nore to anopther
- scheduler assigns (and can be controlled as well) the assignment of pod to node
