# EKS - Create Cluster

## QUICK START

### Cluster UP

Si pas fait, il faut créer une paire de clés nommée kube-demo dans la console EC2 > Paires de clés pour pouvoir ensuite se connecter sur les nodes:

```t
# Créer le cluster EKS à l'aide d'eksctl
$ eksctl create cluster --name=eksdemo1 \
                      --region=eu-west-3 \
                      --zones=eu-west-3a,eu-west-3b \
                      --without-nodegroup

# Créer et associer un fournisseur IAM OIDC pour le cluster EKS
$ eksctl utils associate-iam-oidc-provider \
    --region eu-west-3 \
    --cluster eksdemo1 \
    --approve


# Créer un Groupe de node publique
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
```

Si la politique Amazon_EBS_CSI_Driver n'existe pas (IAM > Politique), on crée le fichier Amazon_EBS_CSI_Driver_Policy.json qui contient :

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



### Cluster DOWN

- Penser à sortir cette nouvelle règle du groupe de sécurité du groupe de nodes publique pour le remote access (sinon cela provoque une erreur car cela ne correspond pas à la stack dans CloudFormation)

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

