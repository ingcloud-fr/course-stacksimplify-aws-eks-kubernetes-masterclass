# Les StatefulSets

Les StatefulSets sont un composant de Kubernetes conçu pour déployer et gérer des applications avec un état persistant. Contrairement aux Deployments (qui gèrent des Pods stateless, sans état), les StatefulSets sont utilisés pour des applications qui nécessitent un stockage persistant, un identifiant réseau stable ou des ordres spécifiques de déploiement et de mise à jour.

## Objectifs et caractéristiques principales des StatefulSets :

- Stockage persistant : Les StatefulSets permettent de garantir que chaque Pod dispose de son propre volume de stockage persistant, qui lui reste dédié même en cas de redémarrage ou de reprogrammation du Pod sur un autre nœud.

- Identité stable des Pods : Chaque Pod géré par un StatefulSet obtient un nom réseau unique et stable, qui reste le même même en cas de redémarrage du Pod. Les Pods sont nommés de manière séquentielle (<nom-du-statefulset>-0, <nom-du-statefulset>-1, etc.). Par exemple, si vous avez un StatefulSet nommé web, les Pods seront nommés web-0, web-1, web-2, etc.

- Ordre de déploiement et de suppression : Les Pods d'un StatefulSet sont déployés dans un ordre précis et sont supprimés dans l'ordre inverse. Cela garantit que chaque Pod démarre et s'arrête en suivant un ordre prédéfini, ce qui est crucial pour des applications qui nécessitent une synchronisation ou un ordre particulier de démarrage.

- Adresse réseau stable : Les StatefulSets créent des Headless Services, ce qui signifie que chaque Pod dispose d’un nom DNS unique. Cela permet aux autres composants de l’application de se connecter de manière stable et prévisible.

## Exemple d'utilisation de StatefulSets :

Les StatefulSets sont généralement utilisés pour les bases de données, les systèmes de messagerie ou les caches distribués, tels que :

- Bases de données : MySQL, PostgreSQL, MongoDB
- Systèmes de messagerie : Kafka, RabbitMQ
- Caches : Redis, Memcached

Comparaison avec les Deployments :

| Caractéristique      | Deployment                                 | StatefulSet                                  |
|:---------------------|:-------------------------------------------|:---------------------------------------------|
| Nom des Pods         | Dynamique, pas de nom fixe                 | Fixe et séquentiel (nom-statefulset-0, etc.) |
| Volumes de stockage  | Non persistants, partagés ou stateless     | Volumes persistants dédiés à chaque Pod      |
| Ordre de déploiement | Pas de garantie sur l'ordre de déploiement | Ordre de déploiement contrôlé                |
| Ordre de suppression | Pas de garantie sur l'ordre de suppression | Ordre de suppression contrôlé                |
| Adresse réseau       | Dynamique                                  | Adresse DNS stable pour chaque Pod           |


## Exemple de StatefulSet YAML

Voici un exemple de **StatefulSet** pour déployer une base de données MySQL :

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql" # Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-persistent-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

Explication du YAML :

- **serviceName** : Spécifie le nom du headless service qui est utilisé pour les adresses réseau des Pods.
- **replicas** : Indique le nombre de réplicas de l'application (ici 3 Pods).
- **template** : Décrit le Pod à créer, comme dans un Deployment.
- **volumeMounts** : Monte un volume pour stocker les données de MySQL dans chaque Pod.
- **volumeClaimTemplates** : Définit un modèle pour créer automatiquement un PersistentVolumeClaim dédié pour chaque Pod du StatefulSet.

## Fonctionnement des StatefulSets :

- Noms des Pods : Les Pods créés par un StatefulSet ont des noms stables. Dans l'exemple ci-dessus, les noms seraient mysql-0, mysql-1, mysql-2.
- Ordre de déploiement : Les Pods sont déployés séquentiellement (mysql-0 puis mysql-1 puis mysql-2).
- Ordre de suppression : Les Pods sont supprimés dans l'ordre inverse (mysql-2 d'abord, puis mysql-1, etc.).
- Volumes persistants : Chaque Pod obtient son propre volume persistant, qui reste attaché même si le Pod est redémarré ou reprogrammé ailleurs.

## Cas d'utilisation recommandé :

Les StatefulSets sont idéaux pour les applications où chaque instance (ou Pod) doit conserver un état spécifique, avoir des données persistantes et des identités stables. Par exemple :

- Bases de données relationnelles et NoSQL
- Clusters de stockage distribué comme Cassandra ou HDFS
- Systèmes de messagerie comme Kafka ou RabbitMQ
- Caches distribués comme Redis avec réplication maître/esclave

## Conclusion

Les StatefulSets offrent une solution fiable pour déployer des applications avec des exigences spécifiques en termes de persistance des données, d'identité stable et d'ordre de déploiement. Ils sont une composante essentielle de Kubernetes pour les applications nécessitant une gestion précise de l'état.