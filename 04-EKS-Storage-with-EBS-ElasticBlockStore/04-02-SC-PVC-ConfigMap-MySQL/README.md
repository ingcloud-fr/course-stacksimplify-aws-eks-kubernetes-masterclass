# EKS Storage -  Storage Classes, Persistent Volume Claims

## Step-01: Introduction
- We are going to create a MySQL Database with persistence storage using AWS EBS Volumes

| Kubernetes Object  | YAML File |
| ------------- | ------------- |
| Storage Class  | 01-storage-class.yml |
| Persistent Volume Claim | 02-persistent-volume-claim.yml   |
| Config Map  | 03-UserManagement-ConfigMap.yml  |
| Deployment, Environment Variables, Volumes, VolumeMounts  | 04-mysql-deployment.yml  |
| ClusterIP Service  | 05-mysql-clusterip-service.yml  |

## Step-02: Create following Kubernetes manifests
### Create Storage Class manifest
- https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode
- **Important Note:** `WaitForFirstConsumer` mode will delay the volume binding and provisioning  of a PersistentVolume until a Pod using the PersistentVolumeClaim is created. 


**01-storage-class.yml**

```t
apiVersion: storage.k8s.io/v1   # Spécifie la version de l'API utilisée pour définir une StorageClass.
kind: StorageClass              # Type de ressource Kubernetes utilisée pour gérer les types de stockage.
metadata: 
  name: ebs-sc                  # Nom de la StorageClass. Il sera utilisé pour référencer cette classe de stockage dans les PVC (Persistent Volume Claims).
provisioner: ebs.csi.aws.com    # Spécifie le provisioner (fournisseur de stockage) à utiliser. Ici, "ebs.csi.aws.com" indique l'utilisation du CSI (Container Storage Interface) pour gérer les volumes EBS sur AWS.
volumeBindingMode: WaitForFirstConsumer  # Définie le mode de liaison des volumes. "WaitForFirstConsumer" signifie que le volume ne sera provisionné qu'une fois qu'il y aura un Pod consommateur, garantissant ainsi que le volume soit créé dans la zone de disponibilité où le Pod s'exécute.
```

Explication détaillée des champs :

- **apiVersion** : Définit la version de l'API Kubernetes utilisée pour gérer cette ressource. Ici, on utilise storage.k8s.io/v1 pour les StorageClass.

- **kind** : Indique le type de ressource, ici StorageClass, qui est utilisée pour définir la manière dont Kubernetes doit provisionner les volumes de stockage.

- **metadata.name** : Nom unique de cette StorageClass. Ce nom est utilisé pour référencer cette classe lorsque l'on définit un PVC (Persistent Volume Claim).

- **provisioner** : Définit le plugin CSI ou provisioner qui sera utilisé pour créer des volumes persistants. ebs.csi.aws.com est le provisioner fourni par AWS pour créer des volumes EBS.

- **volumeBindingMode** : Définit quand le volume sera lié (attaché). WaitForFirstConsumer signifie que le volume ne sera créé qu'au moment où un Pod qui le consomme sera déployé. Cela permet de s'assurer que le volume EBS est créé dans la même zone de disponibilité que le Pod qui le consomme.

En résumé, cette configuration de StorageClass crée une classe de stockage qui utilise le provisioner CSI EBS d'AWS et attend qu'un Pod demande un volume avant de le créer, garantissant une bonne correspondance de la zone de disponibilité.



### Create Persistent Volume Claims manifest

**02-persistent-volume-claim.yml**

```t
apiVersion: v1                # Spécifie la version de l'API utilisée pour définir une PersistentVolumeClaim (PVC).
kind: PersistentVolumeClaim   # Type de ressource Kubernetes utilisée pour réclamer un volume persistant.
metadata:
  name: ebs-mysql-pv-claim    # Nom unique de la PersistentVolumeClaim (PVC). Ce nom est utilisé pour référencer cette PVC dans les Pods ou Deployments.
spec: 
  accessModes:
    - ReadWriteOnce           # Spécifie le mode d'accès au volume. "ReadWriteOnce" signifie que le volume peut être monté en lecture/écriture par un seul Pod à la fois.
  storageClassName: ebs-sc    # Indique le nom de la StorageClass à utiliser pour provisionner le volume. Ici, on utilise la StorageClass "ebs-sc" définie précédemment.
  resources: 
    requests:
      storage: 4Gi            # Spécifie la quantité de stockage demandée pour le volume. Ici, la PVC demande 4 GiB de stockage.

```
Explication détaillée des champs :

- **apiVersion** : Définit la version de l'API Kubernetes utilisée pour gérer cette ressource. Ici, on utilise v1 pour les PersistentVolumeClaim.

- **kind** : Indique le type de ressource, ici PersistentVolumeClaim (PVC), qui est utilisée pour réclamer un volume persistant dans Kubernetes.

- **metadata.name** : Nom unique attribué à la PVC. Ce nom est utilisé pour faire référence à ce volume persistant dans les spécifications de Pods ou de Déploiements.

- **spec** : Spécifie les détails de la PVC.
-- **accessModes** : Définit le mode d'accès au volume. ReadWriteOnce signifie que le volume peut être monté en lecture/écriture, mais par un seul Pod à la fois. D'autres options incluent ReadOnlyMany (lecture seule par plusieurs Pods) et ReadWriteMany (lecture/écriture par plusieurs Pods).
-- **storageClassName** : Référence la StorageClass définie précédemment, ici "ebs-sc", pour créer un volume de stockage persistant via un fournisseur CSI.
-- **resources.requests.storage** : Définit la quantité de stockage demandée par le volume. Ici, la PVC demande 4 GiB (gibioctets) de stockage.

```
# Create Storage Class & PVC
kubectl apply -f kube-manifests/

# List Storage Classes
kubectl get sc

# List PVC
kubectl get pvc 

# List PV
kubectl get pv
```
### Create ConfigMap manifest
- We are going to create a `usermgmt` database schema during the mysql pod creation time which we will leverage when we deploy User Management Microservice. 

### Create MySQL Deployment manifest
- Environment Variables
- Volumes
- Volume Mounts

### Create MySQL ClusterIP Service manifest
- At any point of time we are going to have only one mysql pod in this design so `ClusterIP: None` will use the `Pod IP Address` instead of creating or allocating a separate IP for `MySQL Cluster IP service`.   

## Step-03: Create MySQL Database with all above manifests
```
# Create MySQL Database
kubectl apply -f kube-manifests/

# List Storage Classes
kubectl get sc

# List PVC
kubectl get pvc 

# List PV
kubectl get pv

# List pods
kubectl get pods 

# List pods based on  label name
kubectl get pods -l app=mysql
```

## Step-04: Connect to MySQL Database
```
# Connect to MYSQL Database
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11

[or]

# Use mysql client latest tag
kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h mysql -pdbpassword11

# Verify usermgmt schema got created which we provided in ConfigMap
mysql> show schemas;
```

## Step-05: References
- We need to discuss references exclusively here. 
- These will help you in writing effective templates based on need in your environments. 
- Few features are still in alpha stage as on today (Example:Resizing), but once they reach beta you can start leveraging those templates and make your trials. 
- **EBS CSI Driver:** https://github.com/kubernetes-sigs/aws-ebs-csi-driver
- **EBS CSI Driver Dynamic Provisioning:**  https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/dynamic-provisioning
- **EBS CSI Driver - Other Examples like Resizing, Snapshot etc:** https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes
- **k8s API Reference Doc:** https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#storageclass-v1-storage-k8s-io


