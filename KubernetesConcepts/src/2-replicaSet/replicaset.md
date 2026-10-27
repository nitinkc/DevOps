there will 3 pods with unique replicas

```shell
kubectl port-forward go-server-replicaset-r5xd5  8090:8080
```

port 8090 is the local host port connected with port 8080 of the container

## create
```shell
kubectl apply -f backend-deployment.yml
```
## Delete 

```shell
kubectl delete -f 3-deloyments.yml
```


## ReplicaSet
- provide template definition of a pod under spec
- under Spec we have selector to manage the pods that were created before the replicat was created
```shell
kubectl create -f <replicat-file-name>.yml
kubectl get replicaset
kubectl get pods
```

Why Label is important : Replicaset is a process that monitors the pod. Labels are needed to keep a group togather

![replicaset_labels.png](../images/replicaset_labels.png)

### Scale the replica
```shell
kubectl delete replicaset myapp-replicaset
kubectl replace -f replicaset-definition.yml
kubectl scale -replicas=6 -f replicaset-definition.yml
```
![scale-replicaset.png](../images/scale-replicaset.png)
