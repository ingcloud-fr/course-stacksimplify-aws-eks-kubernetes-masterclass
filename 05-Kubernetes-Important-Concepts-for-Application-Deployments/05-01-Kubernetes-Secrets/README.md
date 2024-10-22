# Kubernetes - Secrets

## Step-01: Introduction
- Kubernetes Secrets let you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. 
- Storing confidential information in a Secret is safer and more flexible than putting it directly in a Pod definition or in a container image. 

## Step-02: Create Secret for MySQL DB Password
### 
```
# Mac
echo -n 'dbpassword11' | base64

# URL: https://www.base64encode.org
```
### Create Kubernetes Secrets manifest
```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-db-password
#type: Opaque means that from kubernetes's point of view the contents of this Secret is unstructured.
#It can contain arbitrary key-value pairs. 
type: Opaque
data:
  # Output of echo -n 'dbpassword11' | base64
  db-password: ZGJwYXNzd29yZDEx
```
## Step-03: Update secret in MySQL Deployment for DB Password
```yml
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-db-password
                  key: db-password
```

## Step-04: Update secret in UMS Deployment
- UMS means User Management Microservice
```yml
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-db-password
                  key: db-password
```

## Step-05: Create & Test

```t
# Create All Objects
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Access Application Health Status Page
http://<WorkerNode-Public-IP>:31231/usermgmt/health-status
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

## Aller plus loin : kubeseal

**How to Securely Encrypting Secrets in Kubernetes with Kubeseal?** https://medium.com/@rumeysa_25373/how-to-securely-encrypting-secrets-in-kubernetes-with-kubeseal-e96eb6c6d6cd

Pour installer kubeseal dans un cluster Kubernetes, tu dois déployer le Sealed Secrets Controller qui gère le chiffrement et le déchiffrement des secrets. Voici les étapes pour installer kubeseal :

1. Installation du Sealed Secrets Controller avec Helm
La manière la plus simple et recommandée est d'utiliser Helm pour installer le Sealed Secrets Controller.

Ajouter le repository Helm :

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
```

Installer Sealed Secrets dans ton cluster Kubernetes :

```
helm install sealed-secrets sealed-secrets/sealed-secrets
```

Cette commande installera le Sealed Secrets Controller dans le namespace par défaut kube-system. Tu peux spécifier un autre namespace avec -n <nom_du_namespace> si tu le souhaites.

2. Vérifier l'installation
Pour vérifier que le Sealed Secrets Controller est correctement installé et fonctionne, utilise la commande suivante :

```
kubectl get pods -n kube-system | grep sealed-secrets
```

Tu devrais voir un Pod appelé sealed-secrets-controller en état Running.

3. Installation de l'utilitaire kubeseal sur ta machine locale
Le Sealed Secrets Controller a besoin de l'outil CLI kubeseal pour chiffrer les secrets. Tu dois installer cet outil sur ta machine locale.

Pour Linux :

```
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.22.0/kubeseal-linux-amd64
sudo install -m 755 kubeseal-linux-amd64 /usr/local/bin/kubeseal
```

4. Chiffrer un secret avec kubeseal

Une fois le controller installé dans le cluster et kubeseal installé localement, tu peux commencer à chiffrer les secrets. Voici comment chiffrer un secret :

Créer un fichier secret.yaml :

```yml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
data:
  username: YWRtaW4=   # Base64 de "admin"
  password: MWYyZDFlMmU2N2Rm  # Base64 de "1f2d1e2e67df"
```

Chiffrer le secret avec kubeseal :

```
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

Cela va créer un fichier sealed-secret.yaml contenant le secret chiffré. Le fichier sealed-secret.yaml ressemblera à ceci :

```yml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: default
spec:
  encryptedData:
    username: AgBjBJj...  # Longue chaîne chiffrée
    password: AgBlG...    # Longue chaîne chiffrée
```

5. Appliquer le Sealed Secret dans Kubernetes :

```
kubectl apply -f sealed-secret.yaml
```
6. Créer un Déploiement qui utilise ce Secret

Crée un fichier de déploiement, par exemple deployment.yaml :

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp-container
          image: nginx:latest
          env:
            - name: USERNAME
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: username
            - name: PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: password
```