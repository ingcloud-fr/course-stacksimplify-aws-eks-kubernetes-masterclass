# EKS - Create EKS Node Group in Private Subnets

## Step-01: Introduction

- On ne va plus utiliser le service NodePort sur les sous-réseaux publics pour accéder à l'application
- Nous allons créer un groupe de nœuds dans les sous-réseaux privés du VPC.
- Nous allons déployer des charges de travail (workloads) sur le groupe de nœuds privés, où les charges de travail s'exécuteront dans des sous-réseaux privés et un load balancer sera créé dans le sous-réseau public et accessible via Internet.



## Step-02: Delete existing Public Node Group in EKS Cluster

```t
# Get NodeGroups in a EKS Cluster
$ eksctl get nodegroup --cluster=<Cluster-Name>
$ eksctl get nodegroup --cluster=eksdemo1
CLUSTER         NODEGROUP               STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID        ASG NAME    TYPE
eksdemo1        eksdemo1-ng-public1     ACTIVE  2024-10-23T09:26:48Z    2               4               2                       t3.medium       AL2_x86_64      eks-eksdemo1-ng-public1-c6c95c59-9b13-f205-0cf7-cb1687dfe747 managed

# Delete Node Group - Replace nodegroup name and cluster name
eksctl delete nodegroup <NodeGroup-Name> --cluster <Cluster-Name>
eksctl delete nodegroup eksdemo1-ng-public1 --cluster eksdemo1
```

## Step-03: Create EKS Node Group in Private Subnets
- Create Private Node Group in a Cluster
- Key option for the command is `--node-private-networking`

```
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=eu-west-3 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes=2 \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking                       
```

## Step-04: Verify if Node Group created in Private Subnets

### Verify External IP Address for Worker Nodes
- External IP Address should be _<none>_ if our Worker Nodes created in Private Subnets
```
$ kubectl get nodes -o wide
NAME                                           STATUS   ROLES    AGE   VERSION               INTERNAL-IP      EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-192-168-71-63.eu-west-3.compute.internal    Ready    <none>   16m   v1.30.4-eks-a737599   192.168.71.63    <none>        Amazon Linux 2   5.10.226-214.880.amzn2.x86_64   containerd://1.7.22
ip-192-168-96-174.eu-west-3.compute.internal   Ready    <none>   16m   v1.30.4-eks-a737599   192.168.96.174   <none>        Amazon Linux 2   5.10.226-214.880.amzn2.x86_64   containerd://1.7.22

```
### Subnet Route Table Verification - Outbound Traffic goes via NAT Gateway
- Verify the node group subnet routes to ensure it created in private subnets
  - Go to Services -> EKS -> eksdemo -> eksdemo1-ng1-private
  - Click on Associated subnet in **Details** tab
  - Click on **Route Table** Tab.
  - We should see that internet route via NAT Gateway (0.0.0.0/0 -> nat-xxxxxxxx)

