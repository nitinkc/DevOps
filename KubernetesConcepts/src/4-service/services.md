
## Services

helps connect different pods within a node or between multiple nodes
- clusterIP - between pods
- NodePort - from Pod to the outside world
- LoadBalander -

### clusterIp
- metadata: name - Connect the service to the pod
- port: port of the service
- Selector - links the service to the pod
- targetPort

![clusterIpService.png](../images/clusterIpService.png)

### type: NodePort
- ports array
    - targetPort :
    - port :
    - nodePort- 5 digit frm Range [30000 to 32767] : connects pode to outside
- Algorithm: Random

