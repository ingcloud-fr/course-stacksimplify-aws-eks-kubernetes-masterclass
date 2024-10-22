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

## Note sur SC, PVC et PV

En Kubernetes, le StorageClass, la PersistentVolumeClaim (PVC) et le PersistentVolume (PV) sont trois éléments qui fonctionnent ensemble pour gérer le stockage des données de manière flexible et dynamique.

**1. StorageClass : Le modèle de stockage**
Le StorageClass est comme une "recette" ou un "plan" pour créer du stockage. Il décrit le type de stockage que tu veux utiliser et les propriétés associées (comme un disque EBS sur AWS, un disque SSD, etc.). Il détermine aussi la façon dont les volumes vont être créés et gérés (par exemple : taille, type de disque, etc.).

**2. PersistentVolumeClaim (PVC) : La demande de stockage**
Une PVC (PersistentVolumeClaim) est comme une "demande de stockage" faite par une application. C’est une déclaration où l’application dit : "J’ai besoin de tant d’espace et je veux qu’il soit de tel type (via un StorageClass)". La PVC est utilisée par les développeurs ou les administrateurs pour spécifier les besoins de stockage d’une application.

**3. PersistentVolume (PV) : Le stockage alloué**
Le PersistentVolume (PV) est le "vrai espace de stockage" créé ou réservé sur le cluster. Il peut être considéré comme l’espace de stockage réel sur un disque (physique ou virtuel). Le PV est soit créé automatiquement par Kubernetes en fonction des spécifications du StorageClass, soit provisionné manuellement par un administrateur.

### Comment cela fonctionne ensemble ?

- **Définir une StorageClass** : L’administrateur ou l’utilisateur définit une StorageClass qui contient les détails sur le type de stockage à utiliser (par exemple, disques EBS pour AWS ou disques standard pour Google Cloud).

- **Faire une demande de stockage via une PVC** : Un développeur ou une application crée une PVC (PersistentVolumeClaim) pour demander une certaine quantité de stockage (par exemple, 10 GiB). Dans cette PVC, il peut être spécifié d’utiliser une StorageClass particulière.

- **Kubernetes trouve ou crée un PV** : Kubernetes va chercher à satisfaire la demande de la PVC. Si un PV correspondant existe déjà (avec suffisamment d’espace et les bonnes propriétés), Kubernetes l'associera à la PVC. Si aucun PV existant ne correspond, Kubernetes va automatiquement créer un PV en utilisant le modèle de la StorageClass définie.

- **Le PV est associé à la PVC** : Une fois que le PV est créé ou trouvé, il est "lié" à la PVC. L’application peut alors utiliser le stockage alloué par ce PV pour lire ou écrire des données.

### Métaphore simple

- StorageClass : C’est comme un "catalogue de produits" où tu choisis le type de disque ou de stockage que tu veux.

- PVC : C’est comme "faire une commande" en spécifiant combien d’espace tu veux et en choisissant un produit du catalogue (une StorageClass).

- PV : C’est comme "recevoir le produit" une fois la commande traitée.

### En résumé

StorageClass décrit le type de stockage disponible et comment le créer.
PVC est une demande spécifique pour ce stockage, avec une taille et un type.
PV est l’espace de stockage réel alloué en fonction de la demande (PVC) et des spécifications (StorageClass).
Ensemble, ces éléments permettent de gérer le stockage de manière dynamique et efficace dans Kubernetes, en s'assurant que chaque application obtient le stockage dont elle a besoin, avec les bonnes propriétés et de manière automatisée.

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

```t
# Create Storage Class & PVC
kubectl apply -f kube-manifests/02-persistent-volume-claim.yml
```
```t
# List PVC
$ kubectl get pvc 
NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
ebs-mysql-pv-claim   Pending                                      ebs-sc         <unset>                 7s
```
le PVC est en **Pending** car la _ClassStorage_ est en **VolumeBindingMode: WaitForFirstConsumer** : Cela signifie que le volume n'est pas immédiatement créé lorsqu'une PVC est définie (comme on a pu le voir ailleurs pour d'autres _VolumeBindingMode_ ). Au lieu de cela, Kubernetes attend qu'un Pod demande cette PVC pour créer le PV. Cela garantit que le volume est créé dans la bonne zone de disponibilité, là où le Pod est planifié.

```t
# List PV
$ kubectl get pv
No resources found
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
apiVersion: v1       # Spécifie la version de l'API Kubernetes utilisée pour gérer les services.
kind: Service        # Indique que ce fichier décrit un objet de type Service dans Kubernetes.
metadata: 
  name: mysql        # Nom unique attribué au service. Ce nom sera utilisé pour référencer ce service dans le cluster Kubernetes.
spec:
  selector:
    app: mysql       # Indique que ce service sélectionne les Pods ayant le label "app: mysql". Cela signifie que ce service dirigera le trafic vers tous les Pods étiquetés avec "app: mysql".
  ports: 
    - port: 3306     # Définit le port sur lequel le service est exposé au sein du cluster. 3306 est le port standard pour MySQL.
  clusterIP: None    # Spécifie que ce service n'aura pas d'adresse IP de cluster. La valeur "None" signifie que le service fonctionnera comme un *Headless Service*, utilisant les adresses IP des Pods individuels au lieu d'attribuer une IP unique au service. On va utiliser POD IP.
```

#### Explication des sections clés 

- apiVersion et kind : 
  - apiVersion: v1 et kind: Service indiquent que ce fichier définit un service Kubernetes de la version de l'API v1.

- metadata.name : name: 
  - mysql donne un nom unique au service. Ce nom peut être utilisé par d'autres objets Kubernetes (comme des Pods ou des déploiements) pour référencer ce service.

- spec.selector : 
  - selector: app: mysql indique que ce service va sélectionner tous les Pods ayant un label app: mysql. Cela signifie que tout Pod avec cette étiquette sera associé à ce service, et tout le trafic dirigé vers ce service sera acheminé vers les Pods correspondants.

- spec.ports : 
  - ports: - port: 3306 indique que ce service expose le port 3306. Ce port correspond au port sur lequel le service MySQL écoute par défaut.

- clusterIP: None : 
  - La directive clusterIP: None indique que le service ne recevra pas d'adresse IP de cluster. En d'autres termes, ce service est un Headless Service. Les Headless Services sont utilisés lorsqu'on ne veut pas de load balancing intégré, mais qu'on souhaite toujours disposer de la découverte de service DNS fournie par Kubernetes.

  - Dans ce cas, chaque Pod sélectionné par ce service aura une entrée DNS individuelle, ce qui permet à un client de se connecter directement à une instance de Pod spécifique, plutôt qu'à une adresse IP de service partagée.

Pourquoi utiliser un Headless Service ?

Un Headless Service (clusterIP: None) est utile lorsqu'on souhaite interagir directement avec les adresses IP des Pods sélectionnés par le service, sans passer par une adresse IP de service unique qui effectue du load balancing. C'est courant dans les cas où les applications gèrent elles-mêmes le load balancing ou lorsque l'on veut accéder directement aux Pods pour des configurations spécifiques, ici pour des tests avant d'en apprndre plus ...

En résumé :
Ce fichier définit un Service Headless pour les Pods ayant le label app: mysql, exposant le port 3306 (le port par défaut de MySQL). Ce service ne dispose pas d'une adresse IP de cluster unique, mais permet plutôt aux clients de se connecter directement aux adresses IP des Pods individuels qui répondent au label app: mysql. Cela est particulièrement utile pour les applications nécessitant une connexion directe aux Pods, comme dans le cas de bases de données MySQL distribuées ou de clusters de bases de données.

## Step-03: Create MySQL Database with all above manifests

```t
# Create MySQL Database
$ kubectl apply -f kube-manifests/
storageclass.storage.k8s.io/ebs-sc unchanged
persistentvolumeclaim/ebs-mysql-pv-claim unchanged
configmap/usermanagement-dbcreation-script created
deployment.apps/mysql created
service/mysql created

# List Storage Classes
$ kubectl get sc
NAME     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  44m
gp2      kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  3h13m

# List PVC
kubectl get pvc 
$ kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
ebs-mysql-pv-claim   Bound    pvc-a46c2e17-aa41-4460-85d4-4bc7aed5e407   4Gi        RWO            ebs-sc         <unset>                 26m

# List PV
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-a46c2e17-aa41-4460-85d4-4bc7aed5e407   4Gi        RWO            Delete           Bound    default/ebs-mysql-pv-claim   ebs-sc         <unset>                          58s

# List pods
$ kubectl get pods 
NAME                     READY   STATUS    RESTARTS   AGE
mysql-76976dff56-p4xls   1/1     Running   0          78s

# List all based on  label name
kubectl get all -l app=mysql
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-76976dff56-p4xls   1/1     Running   0          108s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-76976dff56   1         1         1       108s
```

### Explications

Ce qui s'est passé lors de la création de ta base de données MySQL implique une interaction entre la _StorageClass_, la _PersistentVolumeClaim_ (PVC) et le _PersistentVolume_ (PV). Voyons pas à pas comment ces éléments ont été provisionnés automatiquement et comment Kubernetes les a orchestrés.

**1. Création de la StorageClass**
Lorsqu'on a créé le fichier YAML pour la _StorageClass_ (ebs-sc), on a défini un provisioner _(ebs.csi.aws.com)_ avec un _VolumeBindingMode_ de _WaitForFirstConsumer_. Cela indique à Kubernetes que les volumes persistants (PV) doivent être créés dynamiquement, à la volée, en utilisant ce provisioner EBS.

StorageClass : Elle définit comment les volumes EBS doivent être créés et gérés.
Provisioner : Dans ton cas, il s'agit du provisioner CSI EBS (Container Storage Interface) d'AWS, qui est responsable de la création des volumes EBS dans AWS.

**2. Création de la PVC (PersistentVolumeClaim)**
Quand on a appliqué au manifeste avec la _PersistentVolumeClaim_ (ebs-mysql-pv-claim), Kubernetes a reçu une demande de stockage de 4Gi en utilisant la StorageClass ebs-sc. À ce moment, Kubernetes doit trouver un PersistentVolume (PV) qui correspond à la demande de la PVC, sinon, il devra en créer un dynamiquement.

Étapes impliquées :
- **Vérification de l'existence d'un PV existant** : Kubernetes vérifie s'il existe déjà un PV disponible qui correspond à la demande de la PVC (même StorageClass, même capacité, même access mode). S'il trouve un PV correspondant, il lie (bind) ce PV à la PVC.

- **Création d'un PV dynamique (provisionnement dynamique)** :

  - Si aucun PV existant ne correspond, Kubernetes demande au provisioner ebs.csi.aws.com de créer un nouveau PV en utilisant la StorageClass ebs-sc.

  - Étant donné que ta StorageClass a été configurée avec le provisioner ebs.csi.aws.com, un nouveau volume EBS de 4Gi a été automatiquement créé dans AWS pour répondre à la demande de la PVC.

  - La **ReclaimPolicy** _Delete_ a été utilisée par défaut pour le PV, ce qui signifie que lorsque tu supprimes la PVC, le PV et le volume EBS seront également supprimés.

- Binding du PV et de la PVC :

  - Une fois le PV créé, Kubernetes lie (bind) automatiquement ce PV à la PVC en attribuant l'identifiant du PV à la PVC.

**3. Création des Pods avec des montages de volumes**

Lors de la création de ton Déploiement MySQL, le Pod a été configuré pour utiliser la PVC comme volume :

- Le Pod utilise un volumeMount pour monter la PVC (ebs-mysql-pv-claim) dans le chemin _/var/lib/mysql_ du conteneur.

- Grâce au mécanisme de binding, le volume réel (PV) est attaché et monté dans le Pod MySQL. Cela permet à MySQL d'utiliser un stockage persistant.

Résumé de l'ordonnancement et du provisionnement :

- StorageClass (ebs-sc) : Définie avec un provisioner CSI EBS et une politique de réclamation Delete.

- PVC (ebs-mysql-pv-claim) : Demande un volume de 4Gi en utilisant la StorageClass ebs-sc.

- Provisionnement dynamique : Kubernetes demande au provisioner CSI de créer un volume EBS en fonction des paramètres de la StorageClass.

- Création d’un PV : Un PV correspondant est créé automatiquement et attaché à la PVC.

- Binding : La PVC et le PV sont liés une fois le PV créé.

- Pod : Le Pod MySQL est créé avec un montage de volume utilisant la PVC, ce qui monte automatiquement le volume réel (EBS) dans le conteneur.

Points importants :

- VolumeBindingMode: WaitForFirstConsumer : Cela signifie que le volume n'est pas immédiatement créé lorsqu'une PVC est définie. Au lieu de cela, Kubernetes attend qu'un Pod demande cette PVC pour créer le PV. Cela garantit que le volume est créé dans la bonne zone de disponibilité, là où le Pod est planifié.
- ReclaimPolicy: Delete : Lorsque la PVC ou le PV est supprimé, le volume EBS sera également supprimé.

Résultat final :

À la fin de cette orchestration :

- On a une _StorageClass_ **ebs-sc** qui est configurée pour provisionner des volumes EBS.
- Une PVC **ebs-mysql-pv-claim** a été créée et liée à un PV qui correspond à la demande de stockage.
- Un Pod MySQL a été créé et utilise cette PVC pour monter un volume persistant.

En utilisant les capacités dynamiques de Kubernetes et du provisioner CSI d'AWS, on a pu automatiquement créer, lier et utiliser un volume persistant pour le déploiement MySQL, sans intervention manuelle dans la gestion des volumes.

## Step-04: Connect to MySQL Database

```t
# Connect to MYSQL Database
$ kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11 mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11
If you don't see a command prompt, try pressing enter.

[or]

# Use mysql client latest tag
kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h mysql -pdbpassword11

# Verify usermgmt schema got created which we provided in ConfigMap
mysql> show schemas;
+---------------------+
| Database            |
+---------------------+
| information_schema  |
| #mysql50#lost+found |
| mysql               |
| performance_schema  |
| usermgmt            |
+---------------------+
5 rows in set (0.00 sec)
```

## Step-05: References
- We need to discuss references exclusively here. 
- These will help you in writing effective templates based on need in your environments. 
- Few features are still in alpha stage as on today (Example:Resizing), but once they reach beta you can start leveraging those templates and make your trials. 
- **EBS CSI Driver:** https://github.com/kubernetes-sigs/aws-ebs-csi-driver
- **EBS CSI Driver Dynamic Provisioning:**  https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/dynamic-provisioning
- **EBS CSI Driver - Other Examples like Resizing, Snapshot etc:** https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes
- **k8s API Reference Doc:** https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#storageclass-v1-storage-k8s-io


