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

1. Type de service NodePort :

- type: NodePort indique que ce service est exposé en tant que NodePort. Cela signifie que Kubernetes ouvrira un port (ici 31231) sur chaque nœud du cluster, permettant d'accéder à l'application depuis l'extérieur du cluster à l'aide de l'IP du nœud et du port spécifié.

2. Sélecteur (selector) :

- selector: app: usermgmt-restapp signifie que ce service dirigera le trafic vers tous les Pods ayant le label app: usermgmt-restapp. Cela garantit que seules les instances spécifiques de cette application recevront le trafic routé via ce service.

3. Définition des ports :

- port: 8095 : C'est le port par lequel le service sera accessible à l'intérieur du cluster. D'autres Pods ou services du cluster peuvent se connecter à ce service en utilisant ce port.

- targetPort: 8095 : C'est le port réel sur lequel le conteneur de l'application écoute. Cela signifie que lorsque le service reçoit des requêtes sur le port 8095, il les redirige vers le port 8095 à l'intérieur des Pods sélectionnés.

- nodePort: 31231 : Kubernetes expose ce service sur le port 31231 de chaque nœud du cluster. Cela permet aux utilisateurs ou applications situées en dehors du cluster d'accéder au service en utilisant l'adresse IP de l'un des nœuds et ce port spécifique.

## Step-03: Create UserManagement Service Deployment & Service 
```
# Create Deployment & NodePort Service
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Verify logs of Usermgmt Microservice pod
kubectl logs -f <Pod-Name>

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```
- **Problem Observation:** 
  - If we deploy all manifests at a time, by the time mysql is ready our `User Management Microservice` pod will be restarting multiple times due to unavailability of Database. 
  - To avoid such situations, we can apply `initContainers` concept to our User management Microservice `Deployment manifest`.
  - We will see that in our next section but for now lets continue to test the application
- **Access Application**
```
# List Services
kubectl get svc

# Get Public IP
kubectl get nodes -o wide

# Access Health Status API for User Management Service
http://<EKS-WorkerNode-Public-IP>:31231/usermgmt/health-status
```

## Step-04: Test User Management Microservice using Postman
### Download Postman client 
- https://www.postman.com/downloads/ 
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
        "email": "dkalyanreddy@gmail.com",
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
```
# Connect to MYSQL Database
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -u root -pdbpassword11

# Verify usermgmt schema got created which we provided in ConfigMap
mysql> show schemas;
mysql> use usermgmt;
mysql> show tables;
mysql> select * from users;
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


