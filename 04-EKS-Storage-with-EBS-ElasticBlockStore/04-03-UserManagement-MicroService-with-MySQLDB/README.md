# Deploy UserManagement Service with MySQL Database


## Step-01: Introduction
- We are going to deploy a **User Management Microservice** which will connect to MySQL Database schema **usermgmt** during startup.
- Then we can test the following APIs
  - Create Users
  - List Users
  - Delete User
  - Health Status 

| Kubernetes Object  | YAML File |
| ------------- | ------------- |
| Deployment, Environment Variables  | 06-UserManagementMicroservice-Deployment.yml  |
| NodePort Service  | 07-UserManagement-Service.yml  |

## Step-02: Create following Kubernetes manifests

### Create User Management Microservice Deployment manifest
- **Environment Variables**

| Key Name  | Value |
| ------------- | ------------- |
| DB_HOSTNAME  | mysql |
| DB_PORT  | 3306  |
| DB_NAME  | usermgmt  |
| DB_USERNAME  | root  |
| DB_PASSWORD | dbpassword11  |  

```t
apiVersion: apps/v1                         # Spécifie la version de l'API utilisée pour gérer les Deployments.
kind: Deployment                           # Indique que ce fichier décrit un déploiement Kubernetes.
metadata:
  name: usermgmt-microservice              # Nom unique attribué au Deployment. Utilisé pour identifier ce déploiement dans le cluster Kubernetes.
  labels:
    app: usermgmt-restapp                 # Label ajouté à l'objet Deployment pour l'identification et le filtrage.
spec:
  replicas: 1                             # Indique le nombre de réplicas à déployer. Ici, une seule instance (Pod) est déployée.
  selector:
    matchLabels:
      app: usermgmt-restapp               # Le sélecteur identifie les Pods gérés par ce déploiement en se basant sur les labels.
  template:  
    metadata:
      labels: 
        app: usermgmt-restapp             # Labels appliqués aux Pods créés par ce Deployment.
    spec:
      containers:
        - name: usermgmt-restapp          # Nom du conteneur dans le Pod.
          image: stacksimplify/kube-usermanagement-microservice:1.0.0  # Image Docker utilisée pour le conteneur, avec la version 1.0.0.
          ports: 
            - containerPort: 8095         # Définit le port exposé par le conteneur. 8095 est le port où l'application écoute.
          env:                            # Définit les variables d'environnement passées au conteneur.
            - name: DB_HOSTNAME
              value: "mysql"              # Nom d'hôte de la base de données. Cela correspond au nom du Service MySQL dans le cluster Kubernetes.
            - name: DB_PORT
              value: "3306"               # Port utilisé pour se connecter à la base de données. 3306 est le port standard pour MySQL.
            - name: DB_NAME
              value: "usermgmt"           # Nom de la base de données à utiliser.
            - name: DB_USERNAME
              value: "root"               # Nom d'utilisateur pour se connecter à la base de données.
            - name: DB_PASSWORD
              value: "dbpassword11"       # Mot de passe pour se connecter à la base de données.

```

#### Explication des sections clés

**1. Déploiement d'un microservice :**

  - Ce déploiement gère un microservice appelé usermgmt-microservice. Un seul réplique est spécifiée, ce qui signifie qu'un Pod unique sera déployé.

**2. Sélection des Pods :**

  - Le sélecteur _matchLabels_ avec le label _app: usermgmt-restapp_ permet d'identifier et de gérer les Pods associés à ce déploiement. Seuls les Pods avec ce label seront gérés par ce déploiement.

**3. Conteneur de l'application :**

  - Un conteneur appelé **usermgmt-restapp** est défini, basé sur l'image Docker **stacksimplify/kube-usermanagement-microservice:1.0.0**. Cela signifie que Kubernetes téléchargera et utilisera cette image depuis un registre Docker pour créer le conteneur.

**4. Configuration des ports :**

  - Le conteneur expose le port 8095, qui est probablement le port sur lequel le microservice écoute pour recevoir des requêtes.

**5. Variables d'environnement :**

  - Les variables d'environnement sont définies pour permettre au microservice de se connecter à une base de données MySQL.
  - DB_HOSTNAME est défini comme "mysql", ce qui correspond probablement au nom du Service MySQL créé précédemment dans le cluster Kubernetes. Cela signifie que le microservice pourra communiquer avec MySQL en utilisant ce nom de service.
  - DB_PORT est défini à 3306, le port standard de MySQL.
  - DB_NAME est défini à "usermgmt", ce qui indique le nom de la base de données à utiliser.
  - DB_USERNAME et DB_PASSWORD sont définis pour autoriser l'accès à la base de données en tant qu'utilisateur root avec le mot de passe spécifié.

Ce Deployment crée un microservice appelé **usermgmt-microservice**, qui utilise l'image Docker **kube-usermanagement-microservice:1.0.0**. Il déploie un seul Pod exposant le **port 8095** et configure des variables d'environnement pour se connecter à une base de données MySQL dans le même cluster Kubernetes. La connexion à la base de données utilise le service **mysql** (référence au nom du service) pour se connecter via l'IP du Pod MySQL.

Ce mécanisme permet au microservice de communiquer avec une base de données MySQL déployée séparément dans le cluster en utilisant les services et la découverte DNS de Kubernetes.

### Create User Management Microservice NodePort Service manifest

- NodePort Service

```t
apiVersion: v1                         # Spécifie la version de l'API Kubernetes utilisée pour gérer les services.
kind: Service                          # Indique que ce fichier décrit un objet de type Service dans Kubernetes.
metadata:
  name: usermgmt-restapp-service       # Nom unique attribué au service. Utilisé pour identifier ce service dans le cluster Kubernetes.
  labels: 
    app: usermgmt-restapp              # Label ajouté au service pour l'identification et le filtrage.
spec:
  type: NodePort                      # Définit le type du service comme "NodePort". Cela expose le service sur un port spécifique de chaque nœud du cluster Kubernetes.
  selector:
    app: usermgmt-restapp             # Sélectionne les Pods ayant le label "app: usermgmt-restapp". Cela permet de diriger le trafic vers ces Pods.
  ports: 
    - port: 8095                      # Définit le port sur lequel le service sera exposé à l'intérieur du cluster. C'est le port sur lequel les autres Pods peuvent accéder à ce service.
      targetPort: 8095                # Spécifie le port du conteneur cible. C'est le port exposé à l'intérieur des Pods sélectionnés (ici, le conteneur de l'application écoute sur 8095).
      nodePort: 31231                 # Spécifie le port sur lequel chaque nœud exposera ce service. Ce port permet d'accéder au service depuis l'extérieur du cluster via l'adresse IP du nœud et ce port.

```

#### Explication des sections clés

**1. Type de service NodePort :**

  - **type: NodePort** indique que ce service est exposé en tant que NodePort. Cela signifie que Kubernetes ouvrira un port (ici 31231) sur chaque nœud du cluster, permettant d'accéder à l'application depuis l'extérieur du cluster à l'aide de l'IP du nœud et du port spécifié.

**2. Sélecteur (selector) :**

  - **selector: app: usermgmt-restapp** signifie que ce service dirigera le trafic vers tous les Pods ayant le label app: usermgmt-restapp. Cela garantit que seules les instances spécifiques de cette application recevront le trafic routé via ce service.

**3. Définition des ports :**

  - **port: 8095** : C'est le port par lequel le service sera accessible à l'intérieur du cluster. D'autres Pods ou services du cluster peuvent se connecter à ce service en utilisant ce port.

  - **targetPort: 8095** : C'est le port réel sur lequel le conteneur de l'application écoute. Cela signifie que lorsque le service reçoit des requêtes sur le port 8095, il les redirige vers le port 8095 à l'intérieur des Pods sélectionnés.

  - **nodePort: 31231** : Kubernetes expose ce service sur le port 31231 de chaque nœud du cluster. Cela permet aux utilisateurs ou applications situées en dehors du cluster d'accéder au service en utilisant l'adresse IP de l'un des nœuds et ce port spécifique.

## Step-03: Create UserManagement Service Deployment & Service 

```t
# Create Deployment & NodePort Service
$ kubectl apply -f kube-manifests/
storageclass.storage.k8s.io/ebs-sc unchanged
persistentvolumeclaim/ebs-mysql-pv-claim unchanged
configmap/usermanagement-dbcreation-script unchanged
deployment.apps/mysql unchanged
service/mysql unchanged
deployment.apps/usermgmt-microservice created
service/usermgmt-restapp-service created

# List Pods
$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
mysql-76976dff56-p4xls                   1/1     Running   0          49m
usermgmt-microservice-694dd64df4-wfj5s   1/1     Running   0          14s

# Verify logs of Usermgmt Microservice pod
$ kubectl logs -f <Pod-Name>
$ kubectl logs -f usermgmt-microservice-694dd64df4-wfj5s
2024-10-22 15:31:14.106  INFO 1 --- [           main] o.h.h.i.QueryTranslatorFactoryInitiator  : HHH000397: Using ASTQueryTranslatorFactory
2024-10-22 15:31:17.168  INFO 1 --- [           main] c.s.r.UserManagementApplication          : Started UserManagementApplication in 15.331 seconds (JVM running for 16.458)
```
On vérifie :

```t
# Verify sc, pvc, pv
$ kubectl get sc,pvc,pv
NAME                                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/ebs-sc   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  96m
storageclass.storage.k8s.io/gp2      kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  4h5m

NAME                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/ebs-mysql-pv-claim   Bound    pvc-a46c2e17-aa41-4460-85d4-4bc7aed5e407   4Gi        RWO            ebs-sc         <unset>                 78m

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-a46c2e17-aa41-4460-85d4-4bc7aed5e407   4Gi        RWO            Delete           Bound    default/ebs-mysql-pv-claim   ebs-sc         <unset>                          52m
```
- **Problem Observation:** 
  - If we deploy all manifests at a time, by the time mysql is ready our `User Management Microservice` pod will be restarting multiple times due to unavailability of Database. 
  - To avoid such situations, we can apply `initContainers` concept to our User management Microservice `Deployment manifest`.
  - We will see that in our next section but for now lets continue to test the application

- **Access Application**
```t
# List Services
$ kubectl get svc
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes                 ClusterIP   10.100.0.1      <none>        443/TCP          4h6m
mysql                      ClusterIP   None            <none>        3306/TCP         53m
usermgmt-restapp-service   NodePort    10.100.102.21   <none>        8095:31231/TCP   4m17s

# Get Public IP
$ kubectl get nodes -o wide
NAME                                           STATUS   ROLES    AGE     VERSION               INTERNAL-IP      EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-192-168-10-171.eu-west-3.compute.internal   Ready    <none>   3h57m   v1.30.4-eks-a737599   192.168.10.171   15.237.126.163   Amazon Linux 2   5.10.226-214.879.amzn2.x86_64   containerd://1.7.22
ip-192-168-54-184.eu-west-3.compute.internal   Ready    <none>   3h57m   v1.30.4-eks-a737599   192.168.54.184   35.180.68.231    Amazon Linux 2   5.10.226-214.879.amzn2.x86_64   containerd://1.7.22

# Access Health Status API for User Management Service
http://<EKS-WorkerNode-Public-IP>:31231/usermgmt/health-status

http://15.237.126.163:31231/usermgmt/health-status
# ou
http://35.180.68.231:31231/usermgmt/health-status
```

## Step-04: Test User Management Microservice using Postman
### Download Postman client 
- https://www.postman.com/downloads/ 

```
$ tar -zxvf postman-linux-x64.tar.gz 
$ cd Postman/
$ ./Postman
```

### Import Project to Postman
- Import the postman project `AWS-EKS-Masterclass-Microservices.postman_collection.json` present in folder `04-03-UserManagement-MicroService-with-MySQLDB`
### Create Environment in postman
- Go to Settings -> Click on Add
- **Environment Name:** UMS-NodePort
  - **Variable:** url
  - **Initial Value:** http://WorkerNode-Public-IP:31231
  - **Current Value:** http://WorkerNode-Public-IP:31231
  - Click on **Add**
### Test User Management Services
- Select the environment before calling any API
- **Health Status API**
  - URL: `{{url}}/usermgmt/health-status`
- **Create User Service**
  - URL: `{{url}}/usermgmt/user`
  - `url` variable will replaced from environment we selected
```json
    {
        "username": "admin1",
        "email": "vincent@gmail.com",
        "role": "ROLE_ADMIN",
        "enabled": true,
        "firstname": "fname1",
        "lastname": "lname1",
        "password": "Pass@123"
    }
```
- **List User Service**
  - URL: `{{url}}/usermgmt/users`

- **Update User Service**
  - URL: `{{url}}/usermgmt/user`
```json
    {
        "username": "admin1",
        "email": "dkalyanreddy@gmail.com",
        "role": "ROLE_ADMIN",
        "enabled": true,
        "firstname": "fname2",
        "lastname": "lname2",
        "password": "Pass@123"
    }
```  
- **Delete User Service**
  - URL: `{{url}}/usermgmt/user/admin1`

## Step-05: Verify Users in MySQL Database
```t
# Connect to MYSQL Database
$ kubectl run
 -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11
If you don't see a command prompt, try pressing enter.

# Verify usermgmt schema got created which we provided in ConfigMap
mysql> show databases;
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

mysql> use usermgmt;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+--------------------+
| Tables_in_usermgmt |
+--------------------+
| users              |
+--------------------+
1 row in set (0.00 sec)

mysql> select * from users;
+------------+-------------------+---------+------------+------------+--------------------------------------------------------------+------------+
| username   | email             | enabled | firstname  | lastname   | password                                                     | role       |
+------------+-------------------+---------+------------+------------+--------------------------------------------------------------+------------+
| microtest1 | vincent@gmail.com |        | MicroFName | MicroLname | $2a$04$IMOpQlTjsFq/ybROn1t/cuCl.TDOnXU4gGzFSKzXRbAxIw16y5oD. | ROLE_ADMIN |
+------------+-------------------+---------+------------+------------+--------------------------------------------------------------+------------+
1 row in set (0.01 sec)

```

## Step-06: Clean-Up
- Delete all k8s objects created as part of this section
```
# Delete All
kubectl delete -f kube-manifests/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```


