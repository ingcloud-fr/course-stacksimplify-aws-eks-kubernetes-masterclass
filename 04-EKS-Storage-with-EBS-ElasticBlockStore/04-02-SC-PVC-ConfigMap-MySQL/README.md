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

On applique :

```
$ kubectl apply -f kube-manifests/01-storage-class.yml 
storageclass.storage.k8s.io/ebs-sc created
```
On liste les StorageClass (sc) ... apparement il y en avait un avant par defaut crée par eksctl : 

```t
$ kubectl get sc
NAME     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  52s
gp2      kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  149m
```
On peut voir les détails de la StorageClass ebs-sc :

```t
$ kubectl describe sc ebs-sc
Name:            ebs-sc
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ebs-sc"},"provisioner":"ebs.csi.aws.com","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           ebs.csi.aws.com
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```
Note pour **ReclaimPolicy:Delete** apparait car c'est ReclaimPolicy par défaut : La ReclaimPolicy qui apparaît (Delete) est celle que le provisioner (ebs.csi.aws.com) utilisera par défaut pour les PV qu'il crée automatiquement à partir de cette StorageClass. Cela signifie que tous les PV provisionnés dynamiquement avec cette StorageClass (ou que n'en auront pas spécifié une autre explicitement car c'est celle par défaut) auront **Delete** comme **ReclaimPolicy**, sauf si on crée manuellement un PV avec une autre ReclaimPolicy. 

La policy Delete indique que lorsque la PVC associée est supprimée, le PersistentVolume PV (et le volume sous-jacent dans le fournisseur de stockage) sera également supprimé. Cela signifie que toutes les données présentes dans le volume seront également supprimées. Cette politique est utile lorsque le volume ne doit pas être conservé une fois que l'application n'en a plus besoin.

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
  - **accessModes** : Définit le mode d'accès au volume. ReadWriteOnce signifie que le volume peut être monté en lecture/écriture, mais par un seul Pod à la fois. D'autres options incluent ReadOnlyMany (lecture seule par plusieurs Pods) et ReadWriteMany (lecture/écriture par plusieurs Pods).
  - **storageClassName** : Référence la StorageClass définie précédemment, ici "ebs-sc", pour créer un volume de stockage persistant via un fournisseur CSI.
  - **resources.requests.storage** : Définit la quantité de stockage demandée par le volume. Ici, la PVC demande 4 GiB (gibioctets) de stockage.

```
# Create Storage Class & PVC
kubectl apply -f kube-manifests/02-persistent-volume-claim.yml

# List PVC
kubectl get pvc 

# List PV
kubectl get pv
```
### Create ConfigMap manifest
- We are going to create a `usermgmt` database schema during the mysql pod creation time which we will leverage when we deploy User Management Microservice. 

```t
apiVersion: v1                        # Spécifie la version de l'API Kubernetes utilisée pour définir un ConfigMap.
kind: ConfigMap                       # Type de ressource Kubernetes utilisée pour stocker des configurations sous forme de paires clé-valeur.
metadata:
  name: usermanagement-dbcreation-script # Nom unique du ConfigMap. Utilisé pour identifier ce ConfigMap dans le cluster Kubernetes.
data: 
  mysql_usermgmt.sql: |-              # Clé "mysql_usermgmt.sql" représentant le nom du fichier ou du script. Le symbole "|-" indique que le contenu suivant est du texte multi-lignes.
    DROP DATABASE IF EXISTS usermgmt;    # Commande SQL pour supprimer une base de données nommée "usermgmt" si elle existe déjà.
    CREATE DATABASE usermgmt;            # Commande SQL pour créer une nouvelle base de données nommée "usermgmt".

```

#### Explications détaillées

- **apiVersion** : Définit la version de l'API Kubernetes utilisée pour gérer cette ressource. Ici, on utilise v1 pour les ConfigMap.

- **kind** : Indique le type de ressource Kubernetes, ici un ConfigMap. Les ConfigMaps sont utilisés pour stocker des données de configuration telles que des fichiers de configuration, des scripts, des paires clé-valeur, etc.

- **metadata.name** : Nom unique attribué à ce ConfigMap. Ce nom est utilisé pour référencer ce ConfigMap dans d'autres ressources Kubernetes, comme des Pods, des Déploiements, ou des Jobs.

- **data** : Section qui contient les paires clé-valeur. Ici, chaque clé représente un "nom de fichier" ou un "nom de script" et la valeur correspond au contenu de ce fichier ou script.
  - **mysql_usermgmt.sql** : La clé indique un fichier ou script SQL appelé mysql_usermgmt.sql.
  - **|-** : Ce symbole indique que le texte suivant est multi-lignes, c'est-à-dire qu'il peut contenir plusieurs lignes de contenu.
  - **DROP DATABASE IF EXISTS usermgmt;** : Commande SQL pour supprimer une base de données nommée usermgmt si elle existe déjà.
  - **CREATE DATABASE usermgmt;** : Commande SQL pour créer une base de données vide nommée usermgmt.

Utilité de ce ConfigMap :

Ce ConfigMap est utile pour stocker un script SQL de création de base de données que tu peux monter dans un Pod ou utiliser avec un conteneur MySQL. Par exemple, ce script peut être utilisé pour initialiser une base de données lorsqu'un Pod démarre ou qu'un Job est exécuté.

En résumé :

Ce ConfigMap contient un script SQL pour gérer une base de données MySQL. Il est identifié par le nom usermanagement-dbcreation-script et contient un fichier appelé mysql_usermgmt.sql avec des instructions pour supprimer et recréer une base de données appelée usermgmt. Cela permet de centraliser et de gérer facilement les scripts SQL de manière configurée dans Kubernetes.


### Create MySQL Deployment manifest
- Environment Variables
- Volumes
- Volume Mounts

```t
apiVersion: apps/v1                           # Spécifie la version de l'API utilisée pour gérer les Deployments.
kind: Deployment                              # Indique que ce fichier décrit un déploiement Kubernetes.
metadata:
  name: mysql                                 # Nom unique du Deployment, utilisé pour identifier le déploiement dans le cluster.
spec: 
  replicas: 1                                 # Indique le nombre de réplicas de ce Deployment. Ici, une seule instance (réplica) est déployée.
  selector:
    matchLabels:                              # Sélectionne les Pods à gérer pour ce déploiement en se basant sur les labels.
      app: mysql                              # Label utilisé pour identifier les Pods gérés par ce Deployment.
  strategy:
    type: Recreate                           # Stratégie de déploiement utilisée. "Recreate" signifie que les anciens Pods sont supprimés avant de recréer de nouveaux Pods. Utile pour éviter les conflits de ressources.
  template: 
    metadata: 
      labels: 
        app: mysql                           # Labels appliqués aux Pods créés par ce Deployment.
    spec: 
      containers:
        - name: mysql                        # Nom du conteneur dans le Pod.
          image: mysql:5.6                   # Image Docker utilisée pour le conteneur. Ici, il s'agit de MySQL version 5.6.
          env:                               # Variables d'environnement passées au conteneur.
            - name: MYSQL_ROOT_PASSWORD      # Définit une variable d'environnement pour le mot de passe root de MySQL.
              value: dbpassword11            # Valeur du mot de passe root pour l'instance MySQL.
          ports:
            - containerPort: 3306            # Port exposé par le conteneur. 3306 est le port standard de MySQL.
              name: mysql                    # Nom donné au port pour identification.
          volumeMounts:
            - name: mysql-persistent-storage # Nom du volume à monter dans le conteneur.
              mountPath: /var/lib/mysql      # Point de montage dans le conteneur pour le stockage des données MySQL.
            - name: usermanagement-dbcreation-script # Monte le ConfigMap contenant le script de création de la base de données.
              mountPath: /docker-entrypoint-initdb.d # Dossier spécifique où MySQL exécute automatiquement les scripts au démarrage.
      volumes: 
        - name: mysql-persistent-storage     # Déclare le volume persistant pour stocker les données MySQL.
          persistentVolumeClaim:
            claimName: ebs-mysql-pv-claim    # Référence la PersistentVolumeClaim qui doit être utilisée pour ce volume.
        - name: usermanagement-dbcreation-script # Déclare un volume basé sur un ConfigMap.
          configMap:
            name: usermanagement-dbcreation-script # Nom du ConfigMap à utiliser pour ce volume.
```

#### Explication des sections clés :

- **Déploiement MySQL** :
  - Ce Deployment spécifie une application MySQL avec une réplique unique. Le nom donné à ce déploiement est mysql.

- **Stratégie de déploiement** :
  - La stratégie Recreate indique que lors d'une mise à jour, les Pods existants seront d'abord supprimés avant que de nouveaux soient créés. Cela peut être nécessaire pour des applications comme les bases de données, afin d'éviter des conflits de données entre les Pods.

- **Conteneur MySQL** :
  - L'image MySQL version 5.6 est utilisée.
  - Une variable d'environnement MYSQL_ROOT_PASSWORD est définie avec un mot de passe pour l'utilisateur root.
  - Le port 3306, qui est le port par défaut de MySQL, est exposé et nommé mysql.

- **VolumeMounts** :
  - Le volume nommé **mysql-persistent-storage** est monté sur _/var/lib/mysql_ pour persister les données de la base de données. C’est le dossier où MySQL stocke ses données par défaut.
  - Le volume nommé **usermanagement-dbcreation-script** est monté sur _/docker-entrypoint-initdb.d_. Ce dossier spécifique est reconnu par l'image Docker MySQL comme un emplacement où les scripts .sql seront automatiquement exécutés au démarrage. Cela permet d'initialiser la base de données avec le script de création défini dans le ConfigMap.

- **Volumes** :

  - **mysql-persistent-storage** : Ce volume fait référence à une PVC (PersistentVolumeClaim) appelée ebs-mysql-pv-claim. Cette PVC est utilisée pour provisionner un volume persistant pour stocker les données MySQL.
  - usermanagement-dbcreation-script : Ce volume fait référence à un ConfigMap contenant le script SQL de création de base de données. Cela permet de charger des scripts ou des fichiers de configuration dans le conteneur au moment du démarrage.

En résumé :

Ce fichier YAML Kubernetes déploie une instance MySQL avec une configuration spécifique :

- Une réplique unique (1 Pod) est créée avec une stratégie de remplacement (Recreate).
- MySQL est configuré pour démarrer avec un mot de passe root spécifique.
- Deux volumes sont montés : un pour le stockage persistant des données MySQL et un autre pour exécuter un script de création de base de données au démarrage, fourni via un ConfigMap.
- Cela garantit que les données MySQL sont persistantes même après un redémarrage et que la base de données est initialisée avec le script fourni lors du démarrage du conteneur.



### Create MySQL ClusterIP Service manifest
- At any point of time we are going to have only one mysql pod in this design so `ClusterIP: None` will use the `Pod IP Address` instead of creating or allocating a separate IP for `MySQL Cluster IP service`.   


```t
```

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

```t
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


