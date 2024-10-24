# EKS - Create Cluster

## QUICK START

### Cluster UP

- Si pas fait, il faut créer une paire de clés nommée kube-demo dans la console EC2 > Paires de clés pour pouvoir ensuite se connecter sur les nodes.

On crée le cluster (sans groupe de nodes) :

```t
# Créer le cluster EKS à l'aide d'eksctl
$ eksctl create cluster --name=eksdemo1 \
                      --region=eu-west-3 \
                      --zones=eu-west-3a,eu-west-3b \
                      --version="1.31" \
                      --without-nodegroup

# Créer et associer un fournisseur IAM OIDC pour le cluster EKS
$ eksctl utils associate-iam-oidc-provider \
    --region eu-west-3 \
    --cluster eksdemo1 \
    --approve


# Créer un Groupe de node public
eksctl create nodegroup --cluster=eksdemo1 \
                       --region=eu-west-3 \
                       --name=eksdemo1-ng-public1 \
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
                       --alb-ingress-access


# Ou un groupe de nodes privé avec l'option --node-private-networking

$ eksctl create nodegroup --cluster=eksdemo1 \
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

#### Si on souhaite utiliser le driver CSI EBS pour monter des volumes persistants

Si la politique Amazon_EBS_CSI_Driver n'existe pas (IAM > Politique), on crée le fichier _Amazon_EBS_CSI_Driver_Policy.json_ qui contient :

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

Et on crée la politique **Amazon_EBS_CSI_Driver** :

```
$ aws iam create-policy --policy-name Amazon_EBS_CSI_Driver --policy-document file://Amazon_EBS_CSI_Driver_Policy.json
```

Obtenir le rôle IAM utilisé par les nœuds de travail et associer cette politique à ce rôle :

```t
# On récupère l'ARN du Rôle du Node Group nommé eksdemo1-ng-public1 (cf création du cluster)
$ aws eks describe-nodegroup --cluster-name eksdemo1 --nodegroup-name eksdemo1-ng-public1 --query "nodegroup.nodeRole" --output text
arn:aws:iam::851725523446:role/eksctl-eksdemo1-nodegroup-eksdemo1-NodeInstanceRole-VdJRDmhl8ue2

# Ou avec une extraction  
$ aws eks describe-nodegroup --cluster-name eksdemo1 --nodegroup-name eksdemo1-ng-public1 --query "nodegroup.nodeRole" --output text | awk -F'/' '{print $2}'
eksctl-eksdemo1-nodegroup-eksdemo1-NodeInstanceRole-VdJRDmhl8ue2

# Recupération de l'ARN de la policy Amazon_EBS_CSI_Driver
$ aws iam list-policies --query "Policies[?PolicyName=='Amazon_EBS_CSI_Driver'].[PolicyName,Arn]" --output text
Amazon_EBS_CSI_Driver   arn:aws:iam::851725523446:policy/Amazon_EBS_CSI_Driver

# On attache la policy Amazon_EBS_CSI_Driver au rôle du Node Group
$ aws iam attach-role-policy --role-name eksctl-eksdemo1-nodegroup-eksdemo1-NodeInstanceRole-VdJRDmhl8ue2 --policy-arn arn:aws:iam::851725523446:policy/Amazon_EBS_CSI_Driver
```

On installe le driver :

```t
# Installer EBS CSI Driver
$ kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

Ajouter une inboud rule dans le groupe de sécurité du groupe de nodes publique pour le remote access
- Nom (ressemble à): **eksctl-eksdemo1-nodegroup-eksdemo1-ng-public1-remoteAccess**
- Ajout de la règle **All Trafic from anywhere** (car les NodePorts sont dynamique au dessus de 30000)

#### Class Storage par défaut

On a une **classe storage gp2** déjà de crée par eksctl :

```t
$ kubectl get sc
NAME   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2    kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  140m

$ kubectl describe sc gp2
Name:            gp2
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"gp2"},"parameters":{"fsType":"ext4","type":"gp2"},"provisioner":"kubernetes.io/aws-ebs","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           kubernetes.io/aws-ebs
Parameters:            fsType=ext4,type=gp2
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

On peut donc l'utiliser au lieu de créer notre propre StorageClass ...

#### IngressClass

Vérifier la présence de la policy AWSLoadBalancerControllerIAMPolicy :

```
$ aws iam list-policies --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy']" --output table
----------------------------------------------------------------------------------------------------------
|                                              ListPolicies                                              |
+--------------------------------+-----------------------------------------------------------------------+
|  Arn                           |  arn:aws:iam::851725523446:policy/AWSLoadBalancerControllerIAMPolicy  |
|  AttachmentCount               |  0                                                                    |
|  CreateDate                    |  2024-10-10T14:06:44+00:00                                            |
|  DefaultVersionId              |  v1                                                                   |
|  IsAttachable                  |  True                                                                 |
|  Path                          |  /                                                                    |
|  PermissionsBoundaryUsageCount |  0                                                                    |
|  PolicyId                      |  ANPA4MTWMDH3KAMTGQ5QP                                                |
|  PolicyName                    |  AWSLoadBalancerControllerIAMPolicy                                   |
|  UpdateDate                    |  2024-10-10T14:06:44+00:00                                            |
+--------------------------------+-----------------------------------------------------------------------+
```

Si elle n'existe pas, on la crée :

On récupère la policy au format json :

$ curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

On la crée dans AWS :

```
$ aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_latest.json
```

On crée dans le cluster un Service Account avec la policy **AWSLoadBalancerControllerIAMPolicy** :

```
$ eksctl create iamserviceaccount \
  --cluster=eksdemo1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::851725523446:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

On vérifie :

```
$ kubectl get sa aws-load-balancer-controller -n kube-system
NAME                           SECRETS   AGE
aws-load-balancer-controller   0         49s
```

Et on peut voir l'ARN du rôle dans le Service Account :

```
$ kubectl describe sa aws-load-balancer-controller -n kube-system
Name:                aws-load-balancer-controller
Namespace:           kube-system
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::851725523446:role/eksctl-eksdemo1-addon-iamserviceaccount-kube--Role1-Mun2PuBFMJhS
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```

On installe le ALB controleur avec heml et le repo eks :

```t
# Installation du repo (si pas déjà fait)
$ helm repo add eks https://aws.github.io/eks-charts
"eks" has been added to your repositories

# Update des repos 
$ helm repo update

# Installation du controleur 
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eksdemo1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=eu-west-3 \
  --set vpcId=vpc-0d9297e4c4ae4766d \
  --set image.repository=602401143452.dkr.ecr.eu-west-3.amazonaws.com/amazon/aws-load-balancer-controller
```


A l'installation avec Heml, on a une IngressClass nommé **alb** :

```t
# Verify IngressClass Resource
$ kubectl get ingressclass
NAME                   CONTROLLER            PARAMETERS   AGE
alb                    ingress.k8s.aws/alb   <none>       70m
```

On peut voir le controlleur utilisé **ingress.k8s.aws/alb** pour le Application LB :

```t
$ kubectl describe ingressclass alb
Name:         alb
Labels:       app.kubernetes.io/instance=aws-load-balancer-controller
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=aws-load-balancer-controller
              app.kubernetes.io/version=v2.9.2
              helm.sh/chart=aws-load-balancer-controller-1.9.2
Annotations:  meta.helm.sh/release-name: aws-load-balancer-controller
              meta.helm.sh/release-namespace: kube-system
Controller:   ingress.k8s.aws/alb
Events:       <none>
```
- Donc pas besoin de créer un IngressClass pour ALB, elle existe déjà 

### Cluster DOWN

- Penser à sortir cette nouvelle règle du groupe de sécurité du groupe de nodes publique pour le remote access (sinon cela provoque une erreur car cela ne correspond pas à la stack dans CloudFormation)
- Supprimer tout ce qui a été crée dans les réseaux : database RDS, groupe de sous-réseaux RDS, etc.

```t
# List EKS Clusters
eksctl get clusters

# Capture Node Group name
$ eksctl get nodegroup --cluster=<clusterName>
$ eksctl get nodegroup --cluster=eksdemo1
CLUSTER         NODEGROUP               STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID      ASG NAME                      TYPE
eksdemo1        eksdemo1-ng-public1     ACTIVE  2024-10-22T11:37:13Z    2               4               2                       t3.medium       AL2_x86_64    eks-eksdemo1-ng-public1-5ec95a02-21fb-9b03-c996-2ab1498e0a94     managed

# Delete Node Group
eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName>
eksctl delete nodegroup --cluster=eksdemo1 --name=eksdemo1-ng-public1
```

En cas d'erreur de pods inévincable :

```
$ eksctl delete nodegroup eksdemo1-ng-public1 --cluster eksdemo1
2024-10-23 16:42:51 [ℹ]  1 nodegroup (eksdemo1-ng-public1) was included (based on the include/exclude rules)
2024-10-23 16:42:51 [ℹ]  will drain 1 nodegroup(s) in cluster "eksdemo1"
2024-10-23 16:42:51 [ℹ]  starting parallel draining, max in-flight of 1
2024-10-23 16:42:53 [ℹ]  cordon node "ip-192-168-26-24.eu-west-3.compute.internal"
2024-10-23 16:42:53 [ℹ]  cordon node "ip-192-168-49-205.eu-west-3.compute.internal"
2024-10-23 16:44:15 [!]  2 pods are unevictable from node ip-192-168-26-24.eu-west-3.compute.internal
2024-10-23 16:45:23 [!]  2 pods are unevictable from node ip-192-168-26-24.eu-west-3.compute.internal
2024-10-23 16:46:31 [!]  2 pods are unevictable from node ip-192-168-26-24.eu-west-3.compute.internal
```

On force l'éviction des pods avec l'option **--disable-eviction** :

```
$ eksctl delete nodegroup eksdemo1-ng-public1 --cluster eksdemo1 --disable-eviction
2024-10-23 16:50:01 [ℹ]  1 nodegroup (eksdemo1-ng-public1) was included (based on the include/exclude rules)
2024-10-23 16:50:01 [ℹ]  will drain 1 nodegroup(s) in cluster "eksdemo1"
2024-10-23 16:50:01 [ℹ]  starting parallel draining, max in-flight of 1
2024-10-23 16:50:14 [✔]  drained all nodes: [ip-192-168-26-24.eu-west-3.compute.internal ip-192-168-49-205.eu-west-3.compute.internal]
2024-10-23 16:50:14 [✖]  failed to acquire semaphore while waiting for all routines to finish: context canceled
2024-10-23 16:50:14 [ℹ]  will delete 1 nodegroups from cluster "eksdemo1"
2024-10-23 16:50:15 [ℹ]  1 task: { 1 task: { delete nodegroup "eksdemo1-ng-public1" [async] } }
2024-10-23 16:50:15 [ℹ]  will delete stack "eksctl-eksdemo1-nodegroup-eksdemo1-ng-public1"
2024-10-23 16:50:15 [✔]  deleted 1 nodegroup(s) from cluster "eksdemo1"
```

On vérifie que le groupe de nodes est bien effacé.

Là il est toujours en effacement (DELETING) :

```
$ eksctl get nodegroup --cluster eksdemo1
CLUSTER         NODEGROUP               STATUS          CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID        ASG NAME                                                     TYPE
eksdemo1        eksdemo1-ng-public1     DELETING        2024-10-23T09:26:48Z    2               4               2                       t3.medium       AL2_x86_64      eks-eksdemo1-ng-public1-c6c95c59-9b13-f205-0cf7-cb1687dfe747 managed
```
Au bout de quelques minutes :

```
$ eksctl get nodegroup --cluster eksdemo1
Error: No nodegroups found
```

Maintenant on peut supprimer le cluster :

```
# Delete Cluster
eksctl delete cluster <clusterName>
eksctl delete cluster eksdemo1
```

## List of Topics 
- Install CLIs
  - AWS CLI
  - kubectl
  - eksctl
- Create EKS Cluster
- Create EKS Node Groups
- Understand EKS Cluster Pricing
  - EKS Control Plane
  - EKS Worker Nodes
  - EKS Fargate Profile
- Delete EKS Clusters 

