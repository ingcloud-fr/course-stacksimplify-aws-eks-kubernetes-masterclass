# Kubernetes - Requests and Limits

## Step-01: Introduction
- We can specify how much each container a pod needs the resources like CPU & Memory. 
- When we provide this information in our pod, the scheduler uses this information to decide which node to place the Pod on. 
- When you specify a resource limit for a Container, the kubelet enforces those `limits` so that the running container is not allowed to use more of that resource than the limit you set. 
-  The kubelet also reserves at least the `request` amount of that system resource specifically for that container to use.
- Si pas de ressources sur les nodes, le pod passe en pending.
- Peu utile dans le cas de l'autoscaling.
- Ne sera pas mis dans les prochains templates pour ces raisons.

## Step-02: Add Requests & Limits
```yml
          resources:
            requests:
              memory: "128Mi" # 128 MebiByte is equal to 135 Megabyte (MB)
              cpu: "500m" # `m` means milliCPU
            limits:
              memory: "500Mi"
              cpu: "1000m"  # 1000m is equal to 1 VCPU core                                          
```

## Step-03: Create k8s objects & Test
```t
# Create All Objects
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Watch List Pods screen
kubectl get pods -w

# Describe Pod & Discuss about init container
kubectl describe pod <usermgmt-microservice-xxxxxx>

# Access Application Health Status Page
http://<WorkerNode-Public-IP>:31231/usermgmt/health-status

# List Nodes & Describe Node
kubectl get nodes
kubectl describe node <Node-Name>
```

On peut vérifier la distribution des ressources dans les nodes :

```t
$ kubectl get nodes
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-26-24.eu-west-3.compute.internal    Ready    <none>   89m   v1.30.4-eks-a737599
ip-192-168-49-205.eu-west-3.compute.internal   Ready    <none>   89m   v1.30.4-eks-a737599

$ kubectl describe node ip-192-168-26-24.eu-west-3.compute.internal
Name:               ip-192-168-26-24.eu-west-3.compute.internal
...
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Wed, 23 Oct 2024 12:56:57 +0200   Wed, 23 Oct 2024 11:27:58 +0200   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Wed, 23 Oct 2024 12:56:57 +0200   Wed, 23 Oct 2024 11:27:58 +0200   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Wed, 23 Oct 2024 12:56:57 +0200   Wed, 23 Oct 2024 11:27:58 +0200   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Wed, 23 Oct 2024 12:56:57 +0200   Wed, 23 Oct 2024 11:28:10 +0200   KubeletReady                 kubelet is posting ready status
Addresses:
  ...
Capacity:
  cpu:                2
  ephemeral-storage:  20959212Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3943312Ki
  pods:               17
Allocatable:
  cpu:                1930m
  ephemeral-storage:  18242267924
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3388304Ki
  pods:               17
...
Non-terminated Pods:          (7 in total)
  Namespace                   Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                  ------------  ----------  ---------------  -------------  ---
  dev2                        mysql-64864d79c7-x4qrk                0 (0%)        0 (0%)      0 (0%)           0 (0%)         50m
  kube-system                 aws-node-56htn                        50m (2%)      0 (0%)      0 (0%)           0 (0%)         90m
  kube-system                 coredns-cc6ccd49c-5j2jr               100m (5%)     0 (0%)      70Mi (2%)        170Mi (5%)     96m
  kube-system                 coredns-cc6ccd49c-nnqx7               100m (5%)     0 (0%)      70Mi (2%)        170Mi (5%)     96m
  kube-system                 ebs-csi-controller-89bc955fc-gvhjx    60m (3%)      0 (0%)      240Mi (7%)       1536Mi (46%)   22m
  kube-system                 ebs-csi-node-dnf7n                    30m (1%)      0 (0%)      120Mi (3%)       768Mi (23%)    22m
  kube-system                 kube-proxy-tnd9f                      100m (5%)     0 (0%)      0 (0%)           0 (0%)         90m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                440m (22%)   0 (0%)
  memory             500Mi (15%)  2644Mi (79%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
Events:              <none>

```


## Step-04: Clean-Up
- Delete all k8s objects created as part of this section
```
# Delete All
kubectl delete -f kube-manifests/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```

## References:
- https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/