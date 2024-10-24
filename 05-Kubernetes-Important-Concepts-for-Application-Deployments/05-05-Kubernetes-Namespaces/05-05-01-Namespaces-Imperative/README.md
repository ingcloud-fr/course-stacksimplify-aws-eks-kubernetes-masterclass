# Kubernetes Namespaces - Imperative using kubectl

## Step-01: Introduction
- Namespaces allow to split-up resources into different groups.
- Resource names should be unique in a namespace
- We can use namespaces to create multiple environments like dev, staging and production etc
- Kubernetes will always list the resources from `default namespace` unless we provide exclusively from which namespace we need information from.

## Step-02: Namespaces Generic - Deploy in Dev1 and Dev2
### Create Namespace
```t
# List Namespaces
kubectl get ns 

# Create Namespace
$ kubectl create namespace <namespace-name>
$ kubectl create namespace dev1
namespace/dev2 created
$ kubectl create namespace dev2
namespace/dev2 created

# List Namespaces
kubectl get ns 
```
### Comment NodePort in UserMgmt NodePort Service
- **File: 07-UserManagement-Service.yml**
- **Why?:**
  - Whenever we create with same manifests multiple environments like dev1, dev2 with namespaces, we cannot have same worker node port for multiple services. 
  - We will have port conflict on the nodes. 
  - Its good for k8s system to provide dynamic nodeport for us in such situations.
  - Donc on commente **ports.nodePort: 31231** dans 07-UserManagement-Service.yml pour avoir une attribution dynamique du NodePort

```yml
      #nodePort: 31231
```
- **Error** if not commented
```log
The Service "usermgmt-restapp-service" is invalid: spec.ports[0].nodePort: Invalid value: 31231: provided port is already allocated
```
### Deploy All k8s Objects
```t
# Deploy All k8s Objects
$ kubectl apply -f kube-manifests/ -n dev1
storageclass.storage.k8s.io/ebs-sc created
persistentvolumeclaim/ebs-mysql-pv-claim created
configmap/usermanagement-dbcreation-script created
deployment.apps/mysql created
service/mysql created
deployment.apps/usermgmt-microservice created
service/usermgmt-restapp-service created
secret/mysql-db-password created

$ kubectl apply -f kube-manifests/ -n dev2

# List all objects from dev1 & dev2 Namespaces
$ kubectl get all -n dev1
NAME                                        READY   STATUS    RESTARTS   AGE
pod/mysql-64864d79c7-fnr7z                  1/1     Running   0          32m
pod/usermgmt-microservice-fc67b4fbc-ntmfj   1/1     Running   0          32m

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/mysql                      ClusterIP   None             <none>        3306/TCP         32m
service/usermgmt-restapp-service   NodePort    10.100.192.182   <none>        8095:32152/TCP   32m

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql                   1/1     1            1           32m
deployment.apps/usermgmt-microservice   1/1     1            1           32m

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-64864d79c7                  1         1         1       32m
replicaset.apps/usermgmt-microservice-fc67b4fbc   1         1         1       32m

$ kubectl get all -n dev2
...
```
## Step-03: Verify SC,PVC and PV
- **Shorter Note**
  - PVC is a **namespace specific resource**
  - PV and SC are **generic**

- **Observation-1:** `Persistent Volume Claim (PVC)` gets created in respective namespaces

```t
# List PVC for dev1 and dev2
$ kubectl get pvc -n dev1
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
ebs-mysql-pv-claim   Bound    pvc-7d97f6b3-84dd-4c15-aadf-68a64e253a55   4Gi        RWO            ebs-sc         <unset>                 33m

$ kubectl get pvc -n dev2
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
ebs-mysql-pv-claim   Bound    pvc-313c2330-d7a4-4bcb-b5d7-c8ade64dcaaa   4Gi        RWO            ebs-sc         <unset>                 33m
```

- **Observation-2:** `Storage Class (SC) and Persistent Volume (PV)` gets created generic. No specifc namespace for them   

```t
# List sc,pv
$ kubect get sc,pv

$ kubectl get sc
NAME     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  34m
gp2      kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  83m

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-313c2330-d7a4-4bcb-b5d7-c8ade64dcaaa   4Gi        RWO            Delete           Bound    dev2/ebs-mysql-pv-claim   ebs-sc <unset>                          6m19s
pvc-7d97f6b3-84dd-4c15-aadf-68a64e253a55   4Gi        RWO            Delete           Bound    dev1/ebs-mysql-pv-claim   ebs-sc         <unset>                          6m21s
```
## Step-04: Access Application
### Dev1 Namespace
```t
# Get Public IP
$ kubectl get nodes -o wide
NAME                                           STATUS   ROLES    AGE   VERSION               INTERNAL-IP      EXTERNAL-IP     OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-192-168-26-24.eu-west-3.compute.internal    Ready    <none>   75m   v1.30.4-eks-a737599   192.168.26.24    13.38.103.250   Amazon Linux 2   5.10.226-214.880.amzn2.x86_64   containerd://1.7.22
ip-192-168-49-205.eu-west-3.compute.internal   Ready    <none>   75m   v1.30.4-eks-a737599   192.168.49.205   15.237.58.28    Amazon Linux 2   5.10.226-214.880.amzn2.x86_64   containerd://1.7.22

# Get NodePort for dev1 usermgmt service
$ kubectl get svc -n dev1
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mysql                      ClusterIP   None             <none>        3306/TCP         36m
usermgmt-restapp-service   NodePort    10.100.192.182   <none>        8095:32152/TCP   36m

# Access Application
http://<Worker-Node-Public-Ip>:<Dev1-NodePort>/usermgmt/health-status
http://13.38.103.250:32152/usermgmt/health-status

```
### Dev2 Namespace
```t
# Get Public IP
$ kubectl get nodes -o wide

# Get NodePort for dev2 usermgmt service
$ kubectl get svc -n dev2
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mysql                      ClusterIP   None             <none>        3306/TCP         37m
usermgmt-restapp-service   NodePort    10.100.218.174   <none>        8095:32722/TCP   37m

# Access Application
http://<Worker-Node-Public-Ip>:<Dev2-NodePort>/usermgmt/health-status
http://13.38.103.250:32722/usermgmt/health-status
```

## Step-05: Clean-Up

```t
# Delete namespaces dev1 & dev2
$ kubectl delete ns dev1
namespace "dev1" deleted

$ kubectl delete ns dev2
namespace "dev2" deleted

# List all objects from dev1 & dev2 Namespaces
$ kubectl get all -n dev1
No resources found in dev1 namespace.

$ kubectl get all -n dev2
No resources found in dev2 namespace.

# List Namespaces
$ kubectl get ns
NAME              STATUS   AGE
default           Active   110m
kube-node-lease   Active   110m
kube-public       Active   110m
kube-system       Active   110m

# List sc,pv
$ kubectl get sc
NAME     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  62m
gp2      kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  111m

$ kubectl get pv
No resources found

# Delete Storage Class
$ kubectl delete sc ebs-sc
storageclass.storage.k8s.io "ebs-sc" deleted

# Get all from All Namespaces
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   pod/aws-node-56htn                       2/2     Running   0          102m
kube-system   pod/aws-node-ldcw9                       2/2     Running   0          102m
kube-system   pod/coredns-cc6ccd49c-5j2jr              1/1     Running   0          109m
kube-system   pod/coredns-cc6ccd49c-nnqx7              1/1     Running   0          109m
kube-system   pod/ebs-csi-controller-89bc955fc-fdcf5   6/6     Running   0          35m
kube-system   pod/ebs-csi-controller-89bc955fc-gvhjx   6/6     Running   0          35m
kube-system   pod/ebs-csi-node-7qvzx                   3/3     Running   0          35m
kube-system   pod/ebs-csi-node-dnf7n                   3/3     Running   0          35m
kube-system   pod/kube-proxy-tblfx                     1/1     Running   0          102m
kube-system   pod/kube-proxy-tnd9f                     1/1     Running   0          102m

NAMESPACE     NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.100.0.1    <none>        443/TCP                  112m
kube-system   service/kube-dns     ClusterIP   10.100.0.10   <none>        53/UDP,53/TCP,9153/TCP   109m

NAMESPACE     NAME                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/aws-node       2         2         2       2            2           <none>                   109m
kube-system   daemonset.apps/ebs-csi-node   2         2         2       2            2           kubernetes.io/os=linux   35m
kube-system   daemonset.apps/kube-proxy     2         2         2       2            2           <none>                   109m

NAMESPACE     NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns              2/2     2            2           109m
kube-system   deployment.apps/ebs-csi-controller   2/2     2            2           35m

NAMESPACE     NAME                                           DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-cc6ccd49c              2         2         2       109m
kube-system   replicaset.apps/ebs-csi-controller-89bc955fc   2         2         2       35m
```

## References:
- https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/