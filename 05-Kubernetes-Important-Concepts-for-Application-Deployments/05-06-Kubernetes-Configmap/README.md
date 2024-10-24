# Qu'est ce qu'une ConfigMap Kubernetes ?

Les *ConfigMaps* dans Kubernetes sont des objets qui permettent de stocker des données de configuration **sous forme de paires clé-valeur**. Ils permettent de séparer les configurations d’une application du code applicatif, facilitant ainsi la gestion, la mise à jour et le déploiement des applications dans un cluster Kubernetes.

# Objectif et rôle des ConfigMaps

- **Séparation de la configuration et du code** : Permet de gérer les configurations de manière indépendante du code de l’application.
- **Facilité de mise à jour** : Les ConfigMaps permettent de changer les configurations sans devoir reconstruire et redéployer les conteneurs.
- **Partage de configurations** : Les ConfigMaps peuvent être partagés entre différents Pods au sein d'un cluster.

# Création d'un ConfigMap

Les ConfigMaps peuvent être créés de plusieurs manières :

- **À partir d'un fichier YAML** : Cette méthode est la plus courante. Un fichier YAML permet de définir les paires clé-valeur de manière déclarative.

- **À partir d'un fichier de configuration ou d’un répertoire** : Vous pouvez créer un ConfigMap directement à partir d'un fichier de configuration existant.

À partir de la ligne de commande avec **kubectl** : Vous pouvez créer rapidement un ConfigMap avec des paires clé-valeur.

# Exemple de ConfigMap

Voici un exemple de ConfigMap créé via un fichier YAML :

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  app.properties: |
    server.port=8080
    management.endpoint.health.enabled=true
  log_level: "DEBUG"
  app_name: "My Application"
```

## Explication des champs

- **apiVersion** : La version de l’API Kubernetes pour cet objet. Les ConfigMaps utilisent v1.
- **kind** : Le type de ressource, ici ConfigMap.
- **metadata** : Informations sur l'objet, notamment son nom (name).
- **data** : Une section contenant les données de configuration sous forme de paires clé-valeur. Dans cet exemple, il y a deux types de données :
  - Une entrée multi-lignes app.properties pour des configurations complexes.
  - Des entrées simples telles que log_level et app_name.

# Types de données dans un ConfigMap

- **Clé-Valeur** : Le format de base, où chaque entrée de configuration est une paire clé-valeur.
- **Multi-lignes** : Les fichiers de configuration, tels que les fichiers .properties ou .conf, peuvent être ajoutés directement dans les ConfigMaps en utilisant des blocs de texte.

# Utilisation des ConfigMaps dans les Pods

Les ConfigMaps peuvent être utilisés dans les Pods de deux manières principales :

- **En tant que variables d’environnement** : Vous pouvez injecter des valeurs de ConfigMap sous forme de variables d’environnement dans les conteneurs.

- **En tant que volumes** : Vous pouvez monter un ConfigMap en tant que volume, ce qui permet de rendre les fichiers de configuration disponibles dans un chemin spécifique du conteneur. Cela peut être un répertoire entier ou seulement un fichier.

## 1. Injection de ConfigMap comme variable d’environnement

Voici un exemple de Pod utilisant un ConfigMap en tant que variables d’environnement :

```yml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app-container
      image: my-app:latest
      env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: my-configmap
              key: app_name
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: my-configmap
              key: log_level
```

## 2. Montée d'un ConfigMap en tant que volume (répertoire)

Voici un exemple de Pod montant un ConfigMap en tant que volume :

```yml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app-container
      image: my-app:latest
      volumeMounts:
        - mountPath: /etc/config
          name: config-volume
  volumes:
    - name: config-volume
      configMap:
        name: my-configmap
```

Dans cet exemple, le contenu du ConfigMap my-configmap est monté dans le chemin /etc/config du conteneur. Chaque clé devient un fichier dans ce répertoire, et le contenu de chaque fichier correspond à la valeur de la clé.

## 3. Montée d'un ConfigMap en tant que volume (fichier)

Imaginons que nous souhaitons créer un ConfigMap qui contient un fichier de configuration appelé **app.properties** avec le contenu suivant :

```yml
server.port=8080
management.endpoint.health.enabled=true
```
Voici le fichier YAML pour créer un ConfigMap contenant ce fichier de configuration :

```yml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-configmap
data:
  app.properties: |
    server.port=8080
    management.endpoint.health.enabled=true
```
Voici un exemple de Pod qui monte ce ConfigMap en tant que fichier unique dans le conteneur :

```yml
# pod-with-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app-container
      image: my-app:latest
      volumeMounts:
        - mountPath: /etc/config/app.properties
          subPath: app.properties
          name: config-volume
  volumes:
    - name: config-volume
      configMap:
        name: app-configmap
```
On a :

- **mountPath** : _/etc/config/app.properties_ spécifie le chemin dans le conteneur où le fichier sera monté.
- **subPath** : _app.properties_ indique que seul ce fichier du ConfigMap sera monté. Cela permet de ne pas monter tout le contenu du ConfigMap, mais juste un fichier particulier.
- **name** : Le volume s'appelle _config-volume_ et se réfère à notre ConfigMap app-configmap.


```t
# Créer le ConfigMap 
$ kubectl apply -f configmap.yaml

# Créer le Pod
$ kubectl apply -f pod-with-configmap.yaml
```

# Création de ConfigMap avec kubectl

À partir de la ligne de commande :

```
kubectl create configmap my-configmap --from-literal=app_name="My Application" --from-literal=log_level="DEBUG"
```

À partir d’un fichier de configuration :

```
kubectl create configmap my-configmap --from-file=app.properties
```

À partir d’un répertoire :

```
kubectl create configmap my-configmap --from-file=/path/to/config-dir
```

# Meilleures pratiques avec ConfigMaps

- **Utiliser des noms explicites** : Utilisez des noms de ConfigMaps qui décrivent clairement leur contenu ou leur fonction.
- **Gestion des versions** : Pour des configurations critiques, envisagez de gérer les versions de ConfigMaps en les associant avec des contrôles de version (via Git, par exemple).
- **Séparer la configuration sensible** : Les ConfigMaps ne doivent pas contenir d'informations sensibles (comme des mots de passe ou des clés API). Utilisez plutôt des Secrets pour cela.
- **Automatiser les mises à jour** : Combinez ConfigMaps avec des mécanismes de mise à jour automatique pour vos applications (rolling updates) afin de minimiser les interruptions de service.

# Limites des ConfigMaps

- **Limite de taille** : La taille totale des données dans un ConfigMap est limitée à 1 Mo.
- **Pas sécurisé** : Les ConfigMaps ne sont pas conçus pour stocker des données sensibles. Ils ne sont pas chiffrés comme les Secrets, c'est pourquoi il est recommandé de séparer les configurations sensibles dans des Secrets Kubernetes.

# Conclusion

Les ConfigMaps dans Kubernetes sont essentiels pour la gestion de la configuration des applications. Ils permettent de séparer le code de la configuration, facilitant ainsi le déploiement et la maintenance des applications. Grâce aux ConfigMaps, il est possible de rendre les applications plus flexibles et facilement configurables en fonction des environnements (développement, test, production).

# Exemple avec un reverse proxy nginx

Voici un exemple de configuration complète pour un Pod Nginx en tant que reverse proxy dans Kubernetes. Ce Pod utilisera un ConfigMap pour stocker la configuration Nginx et un Service pour exposer le Pod.

## Objectif

Le but de cette configuration est de créer un Pod avec Nginx qui agit comme un reverse proxy et redirige le trafic vers une application backend ou plusieurs endpoints backend.

## Structure de la configuration

- **ConfigMap** pour la configuration Nginx.
- **Pod avec Nginx** qui utilise la configuration du ConfigMap.
- **Service** pour exposer le Pod.

### Fichier **nginx-configmap.yaml** :

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
      worker_connections 1024;
    }
    http {
      server {
        listen 80;
        
        location / {
          proxy_pass http://backend-service:8080;  # Redirige les requêtes vers le backend
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
        }
      }
    }
```
- Le fichier _nginx.conf_ dans le **ConfigMap** définit un serveur Nginx qui écoute sur le port 80.
- Le bloc _location /_ redirige les requêtes vers un service backend nommé _backend-service_ sur le port 8080.
- Les directives _proxy\_set\_header_ ajoutent des en-têtes pour conserver les informations sur le client initial.

### Fichier **nginx-pod.yaml**

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-reverse-proxy
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          name: nginx-config-volume
  volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config
```

- Le Pod utilise l'image Nginx officielle.
- Il monte le **ConfigMap** _nginx-config_ dans le conteneur Nginx sous le chemin _/etc/nginx/nginx.conf_.
- Le champ **subPath** permet de monter uniquement le fichier **nginx.conf** du ConfigMap dans le Pod.

## Fichier **nginx-service.yaml**


```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-reverse-proxy-service
spec:
  selector:
    app: nginx-reverse-proxy
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort  # Utiliser LoadBalancer si le cluster le supporte

```

- Le Service sélectionne le Pod avec le label **app: nginx-reverse-proxy**.
- Il expose le port 80 du Pod.
- Le type **NodePort** expose le service sur un port spécifique de chaque nœud. Vous pouvez utiliser LoadBalancer si votre cluster supporte les load balancers externes.

```
$ kubectl apply -f nginx-configmap.yaml
$ kubectl apply -f nginx-pod.yaml
$ kubectl apply -f nginx-service.yaml
```

# Pour aller plus loin 

## 1. Gestion des versions des ConfigMaps

Les ConfigMaps stockent des configurations importantes pour les applications. Parfois, ces configurations sont cruciales, et un changement accidentel ou incorrect peut avoir des conséquences graves sur le fonctionnement de l’application. C'est pourquoi il est important de gérer les versions des ConfigMaps de manière appropriée.

### Stratégie de gestion des versions** 

  - **Utilisation d'un dépôt Git** :

      - **Déclarer les ConfigMaps en YAML** : Stockez les ConfigMaps en tant que fichiers YAML dans un dépôt Git, ce qui permet de suivre les modifications, les révisions, et de revenir à une version antérieure si nécessaire.

      - **Commit et révision des changements** : Chaque changement dans les configurations (par exemple, un changement de port, d’URL, de variable d’environnement) doit être consigné sous forme de commit dans Git. Cela permet de conserver un historique clair des modifications apportées.

      - **Gestion des branches** : Utilisez des branches spécifiques dans Git pour tester des configurations avant de les fusionner dans une branche principale ou de production. Cela aide à éviter d'appliquer des configurations non testées.

  - **Versionnage des ConfigMaps dans Kubernetes** :

Bien que Kubernetes ne gère pas explicitement le versionnage des ConfigMaps, vous pouvez utiliser une convention de nommage pour suivre les versions, comme _app-config-v1_, _app-config-v2_, etc. Lors de la mise à jour de la configuration, au lieu de remplacer le ConfigMap existant, créez un nouveau ConfigMap versionné. Cela vous permet de basculer entre différentes versions de ConfigMaps en cas de problème.

Par exemple :

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
data:
  key1: "value1"
Si vous créez une nouvelle version :
```

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
data:
  key1: "new-value"
```

Lors de la mise à jour de vos Pods ou de vos Deployments, vous pouvez spécifier la version du ConfigMap à utiliser, facilitant ainsi un retour arrière rapide en cas de problème.

  - **Stratégie de déploiement GitOps** : En utilisant un outil comme ArgoCD ou Flux, vous pouvez intégrer les ConfigMaps dans une approche GitOps où toutes les configurations sont gérées et versionnées via un dépôt Git. Le déploiement et la synchronisation entre Git et Kubernetes sont automatisés, et les changements de configuration sont contrôlés via des _pull requests_ ou des validations de code.

## 2. Automatiser les mises à jour des ConfigMaps avec des rolling updates

Lorsque vous mettez à jour un _ConfigMap_ dans Kubernetes, cela ne redéploie pas automatiquement les Pods qui l'utilisent. Par conséquent, il est essentiel de mettre en place une stratégie pour éviter les interruptions de service et de déployer les nouvelles configurations de manière fluide.

### Stratégies d'automatisation des mises à jour

  - **Annotation des ConfigMaps** :

Une bonne pratique consiste à utiliser une annotation avec un timestamp ou un identifiant unique dans le Pod ou le Deployment. Cela permet à Kubernetes de détecter un changement dans le ConfigMap et de redéployer les Pods.

Exemple d’annotation dans un Deployment :

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    metadata:
      annotations:
        configmap-reload: "2024-10-21T10:15:00Z"  # Un horodatage changeant lors de chaque mise à jour de ConfigMap
    spec:
      containers:
        - name: my-container
          image: my-image:latest
          envFrom:
            - configMapRef:
                name: my-app-config
```

Lorsque vous mettez à jour le ConfigMap, vous changez également l'annotation configmap-reload avec un nouveau timestamp. Kubernetes détecte ce changement et applique un rolling update des Pods, minimisant les interruptions de service.

  - **Outil de redémarrage automatique**:

Si votre application peut gérer les redémarrages de conteneurs sans perte de données, utilisez des outils comme l'opérateur _Reloader_. Cet opérateur Kubernetes surveille les _ConfigMaps_ et _Secrets_ pour détecter les changements, puis redémarre automatiquement les _Deployments_ ou les _StatefulSets_ qui utilisent ces ConfigMaps.

Pour utiliser Reloader, il suffit d'ajouter une annotation spécifique à votre Deployment :

```yml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

Cela active la détection automatique des changements de ConfigMap par Reloader, ce qui déclenche un redémarrage des Pods concernés.

Mise à jour continue (rolling updates) :

Lors de la mise à jour de votre application ou de votre Pod, assurez-vous que les Pods sont redémarrés de manière progressive (rolling update). Cela signifie que les Pods sont mis à jour un par un, de sorte qu'il y a toujours un Pod en cours d'exécution pour servir les requêtes. Kubernetes prend en charge cette approche via des configurations de stratégie de mise à jour :

```yml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Cela garantit que le nombre de Pods indisponibles à un moment donné est limité, réduisant ainsi l'impact sur les utilisateurs.

En résumé :

- **Gestion des versions des ConfigMaps** : Utilisez Git pour stocker et gérer les versions des ConfigMaps, et envisagez d'utiliser des conventions de nommage ou des stratégies GitOps pour simplifier le retour arrière et le contrôle des modifications.
- **Automatiser les mises à jour** : Combinez des annotations de ConfigMaps, des outils de redémarrage automatique comme Reloader et des stratégies de déploiement progressif pour mettre à jour les configurations sans causer d'interruption de service.

En appliquant ces stratégies, vous améliorez la gestion des configurations critiques et réduisez les risques associés aux changements de configuration dans Kubernetes.

