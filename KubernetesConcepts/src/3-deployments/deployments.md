Think of a ReplicaSet as the engine, and a Deployment as the entire car with an automatic transmission, cruise control, and a reverse gear.

The industry standard. You write Deployments; Kubernetes creates and manages the ReplicaSets underneath

```shell
kubectl apply -f backend-deployment.yml
kubectl port-forward go-server-deployment-7c5ddd7944-fkp4g  8090:8080
kubectl delete -f backend-deployment.yml
```

## two strict Kubernetes rules regarding Deployments

**The Label Mismatch Rule**: A Deployment's `selector.matchLabels` **must be identical** to the labels defined inside its Pod
template (`spec.template.metadata.labels`). The Deployment uses this selector to know which pods belong to it. 

**The Immutable Selector Rule**: Once a Deployment is created in your cluster, its `spec.selector` cannot be changed (it 
is immutable). Because Kubernetes tried to create it once with mismatched or older values, updating it throws an 
immutable field error.

## Deployment

![deployments.png](../images/deployments.png)