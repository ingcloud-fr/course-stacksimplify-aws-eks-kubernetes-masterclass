# Kubernetes - Deployment

## Step-01: Introduction to Deployments
- What is a Deployment?
- What all we can do using Deployment?
- Create a Deployment
- Scale the Deployment
- Expose the Deployment as a Service

Un Deployment est un objet Kubernetes de plus haut niveau qui gère des ReplicaSets et fournit des fonctionnalités supplémentaires, notamment pour les mises à jour progressives et les rollbacks. Lorsque tu crées un Deployment, Kubernetes crée automatiquement un ReplicaSet pour gérer les Pods. Si tu mets à jour l'application, le Deployment crée un nouveau ReplicaSet avec la nouvelle version de l'application et assure une transition progressive entre les deux versions.

Caractéristiques du Deployment :
- Gestion des mises à jour : Le Deployment facilite les mises à jour progressives des Pods (avec une politique de mise à jour rolling ou d'autres stratégies comme blue/green, canary, etc.).
- Rollbacks automatiques : Si une mise à jour échoue, un rollback automatique vers la version précédente est possible.
- Historique des révisions : Le Deployment garde un historique des révisions des ReplicaSets, permettant de revenir à une version antérieure.
- Stratégies de déploiement : Il permet de configurer des stratégies pour déterminer comment les Pods doivent être mis à jour (par exemple, progressivement ou tout en même temps).


## Step-02: Create Deployment
- Create Deployment to rollout a ReplicaSet
- Verify Deployment, ReplicaSet & Pods
- **Docker Image Location:** https://hub.docker.com/repository/docker/stacksimplify/kubenginx
```
# Create Deployment
kubectl create deployment <Deplyment-Name> --image=<Container-Image>
kubectl create deployment my-first-deployment --image=stacksimplify/kubenginx:1.0.0 

# Verify Deployment
kubectl get deployments
kubectl get deploy 

# Describe Deployment
kubectl describe deployment <deployment-name>
kubectl describe deployment my-first-deployment

# Verify ReplicaSet
kubectl get rs

# Verify Pod
kubectl get po
```
## Step-03: Scaling a Deployment
- Scale the deployment to increase the number of replicas (pods)
```
# Scale Up the Deployment
kubectl scale --replicas=20 deployment/<Deployment-Name>
kubectl scale --replicas=20 deployment/my-first-deployment 

# Verify Deployment
kubectl get deploy

# Verify ReplicaSet
kubectl get rs

# Verify Pods
kubectl get po

# Scale Down the Deployment
kubectl scale --replicas=10 deployment/my-first-deployment 
kubectl get deploy
```

## Step-04: Expose Deployment as a Service
- Expose **Deployment** with a service (NodePort Service) to access the application externally (from internet)
```
# Expose Deployment as a Service
kubectl expose deployment <Deployment-Name>  --type=NodePort --port=80 --target-port=80 --name=<Service-Name-To-Be-Created>
kubectl expose deployment my-first-deployment --type=NodePort --port=80 --target-port=80 --name=my-first-deployment-service

# Get Service Info
kubectl get svc
Observation: Make a note of port which starts with 3 (Example: 80:3xxxx/TCP). Capture the port 3xxxx and use it in application URL below. 

# Get Public IP of Worker Nodes
kubectl get nodes -o wide
Observation: Make a note of "EXTERNAL-IP" if your Kubernetes cluster is setup on AWS EKS.
```
- **Access the Application using Public IP**
```
http://<worker-node-public-ip>:<Node-Port>
```