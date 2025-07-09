# Guide de déploiement PostgreSQL HA sur Kubernetes

## 1. Introduction théorique à Kubernetes

### Qu'est-ce que Kubernetes et pourquoi en avons-nous besoin ?

Avant l'avènement de Kubernetes, le déploiement et la gestion des applications étaient confrontés à plusieurs défis majeurs :

**Les problèmes traditionnels :**
- **Gestion manuelle des serveurs** : Les administrateurs devaient manuellement configurer, surveiller et maintenir chaque serveur
- **Scalabilité difficile** : Ajouter de nouvelles instances d'applications nécessitait une intervention manuelle
- **Récupération après panne** : Quand une application tombait en panne, il fallait la redémarrer manuellement
- **Utilisation inefficace des ressources** : Les serveurs étaient souvent sous-utilisés ou surchargés
- **Déploiements complexes** : Mettre à jour une application nécessitait des procédures longues et risquées

**La solution Kubernetes :**

Kubernetes (souvent abrégé K8s) est une plateforme d'orchestration de conteneurs qui automatise le déploiement, la mise à l'échelle et la gestion des applications conteneurisées. Pensez-y comme un "chef d'orchestre" qui coordonne tous les musiciens (vos applications) pour créer une symphonie harmonieuse.

### Concepts fondamentaux de Kubernetes

**1. Cluster Kubernetes**
Un cluster est un ensemble de machines (appelées nœuds) qui travaillent ensemble pour exécuter vos applications. C'est comme avoir une équipe de serveurs qui collaborent.

**2. Pods**
Un Pod est la plus petite unité déployable dans Kubernetes. Il contient un ou plusieurs conteneurs qui partagent le même réseau et stockage. Imaginez un Pod comme un "appartement" où vivent vos applications.

**3. Services**
Un Service permet à vos applications de communiquer entre elles de manière stable, même si les Pods sont recréés. C'est comme avoir un numéro de téléphone fixe qui reste le même même si vous déménagez.

**4. Déploiements (Deployments)**
Un Deployment décrit comment vos applications doivent être déployées et gérées. Il s'assure que le nombre souhaité d'instances de votre application est toujours en fonctionnement.

**5. Volumes persistants**
Pour les applications comme PostgreSQL qui ont besoin de stocker des données de manière permanente, Kubernetes utilise des volumes persistants qui survivent aux redémarrages des Pods.

### Architecture d'un cluster Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│                    CLUSTER KUBERNETES                       │
│                                                             │
│  ┌─────────────────┐              ┌─────────────────────┐   │
│  │   NŒUD MASTER   │              │    NŒUDS WORKERS    │   │
│  │                 │              │                     │   │
│  │ • API Server    │◄────────────►│ • kubelet           │   │
│  │ • etcd          │              │ • kube-proxy        │   │
│  │ • Scheduler     │              │ • Container Runtime │   │
│  │ • Controller    │              │                     │   │
│  └─────────────────┘              └─────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. Outils nécessaires et leurs liens d'installation

### kubectl
kubectl est l'outil en ligne de commande pour interagir avec Kubernetes. C'est votre "télécommande" pour contrôler le cluster.

**Installation :**
- **Documentation officielle** : https://kubernetes.io/docs/tasks/tools/install-kubectl/
- **Windows** : https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
- **macOS** : https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/
- **Linux** : https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

### Minikube
Minikube vous permet de créer un cluster Kubernetes local sur votre machine pour le développement et les tests.

**Installation :**
- **Documentation officielle** : https://minikube.sigs.k8s.io/docs/start/
- **Toutes les plateformes** : https://minikube.sigs.k8s.io/docs/start/

### Helm
Helm est le gestionnaire de paquets pour Kubernetes. Il simplifie l'installation et la gestion d'applications complexes comme PostgreSQL.

**Installation :**
- **Documentation officielle** : https://helm.sh/docs/intro/install/
- **Guide d'installation** : https://helm.sh/docs/intro/install/

## 3. Guide de déploiement PostgreSQL HA sur Kubernetes

### Étape 1 : Vérification de l'environnement

Avant de commencer, assurez-vous que tous les outils sont correctement installés :

```bash
# Vérifier kubectl
kubectl version --client

# Vérifier minikube
minikube version

# Vérifier helm
helm version
```

### Étape 2 : Démarrage du cluster Minikube

Minikube va créer un cluster Kubernetes local sur votre machine :

```bash
# Démarrer minikube avec des ressources suffisantes pour PostgreSQL
minikube start --memory=4096 --cpus=2

# Vérifier que le cluster est en fonctionnement
kubectl get nodes
```

**Explication :** Cette commande crée un cluster Kubernetes local avec 4GB de RAM et 2 CPU. PostgreSQL a besoin de ressources suffisantes pour fonctionner correctement.

### Étape 3 : Configuration de Helm

Helm utilise des "charts" (graphiques) qui sont des modèles pré-configurés pour déployer des applications. Nous devons ajouter le repository Bitnami qui contient le chart PostgreSQL :

```bash
# Ajouter le repository Bitnami (qui contient le chart PostgreSQL HA)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Mettre à jour la liste des charts disponibles
helm repo update

# Vérifier que le repository est bien ajouté
helm repo list
```

**Explication :** Bitnami maintient des charts Helm de haute qualité pour de nombreuses applications populaires, y compris PostgreSQL avec haute disponibilité.

### Étape 4 : Préparation du namespace

Un namespace est comme un "dossier" qui permet d'organiser et d'isoler vos applications dans Kubernetes :

```bash
# Créer un namespace dédié pour PostgreSQL
kubectl create namespace postgres-ha

# Vérifier que le namespace est créé
kubectl get namespaces
```

### Étape 5 : Configuration de PostgreSQL HA

Créons un fichier de configuration pour personnaliser notre déploiement PostgreSQL :

```bash
# Créer un fichier de configuration
cat > postgres-ha-values.yaml << EOF
# Configuration pour PostgreSQL HA
postgresql:
  # Nombre de répliques pour la haute disponibilité
  replicaCount: 3
  
  # Configuration de la base de données
  auth:
    postgresPassword: "motdepasse123"
    database: "ma_base_de_donnees"
    username: "mon_utilisateur"
    password: "motdepasse_utilisateur"
  
  # Configuration des ressources
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

# Configuration du stockage persistant
persistence:
  enabled: true
  size: "10Gi"
  storageClass: "standard"

# Configuration des métriques (optionnel)
metrics:
  enabled: true
  
# Configuration du pgpool pour la répartition de charge
pgpool:
  replicaCount: 2
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "200m"
EOF
```

**Explication détaillée du fichier de configuration :**

- **replicaCount: 3** : Nous créons 3 instances de PostgreSQL pour la haute disponibilité
- **auth** : Configuration des mots de passe et utilisateurs
- **resources** : Limite les ressources CPU et mémoire utilisées par chaque instance
- **persistence** : Active le stockage persistant pour que les données survivent aux redémarrages
- **pgpool** : Configure un pool de connexions pour répartir la charge entre les instances

### Étape 6 : Déploiement de PostgreSQL HA

Maintenant, déployons PostgreSQL avec notre configuration :

```bash
# Déployer PostgreSQL HA avec Helm
helm install postgres-ha bitnami/postgresql-ha \
  --namespace postgres-ha \
  --values postgres-ha-values.yaml

# Vérifier le déploiement
kubectl get pods -n postgres-ha
```

**Explication :** Cette commande utilise Helm pour déployer PostgreSQL HA dans le namespace `postgres-ha` avec notre configuration personnalisée.

### Étape 7 : Vérification du déploiement

Attendez que tous les pods soient en état "Running" :

```bash
# Surveiller le déploiement (Ctrl+C pour arrêter)
kubectl get pods -n postgres-ha -w

# Vérifier les services créés
kubectl get services -n postgres-ha

# Vérifier les volumes persistants
kubectl get pvc -n postgres-ha
```

**Explication :** 
- Les pods doivent être en état "Running"
- Les services permettent d'accéder à PostgreSQL
- Les PVC (Persistent Volume Claims) sont les demandes de stockage pour les données

### Étape 8 : Test de connexion à PostgreSQL

Pour tester notre déploiement, nous allons nous connecter à PostgreSQL :

```bash
# Obtenir le mot de passe de PostgreSQL
export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres-ha postgres-ha-postgresql-ha -o jsonpath="{.data.postgresql-password}" | base64 --decode)

# Se connecter à PostgreSQL via un pod temporaire
kubectl run postgres-client --rm --tty -i --restart='Never' \
  --namespace postgres-ha \
  --image bitnami/postgresql:latest \
  --env="PGPASSWORD=$POSTGRES_PASSWORD" \
  --command -- psql --host postgres-ha-postgresql-ha --port 5432 -U postgres -d ma_base_de_donnees
```

**Explication :** Cette commande crée un pod temporaire avec le client PostgreSQL pour tester la connexion.

### Étape 9 : Exposition du service (optionnel)

Pour accéder à PostgreSQL depuis l'extérieur du cluster :

```bash
# Créer un service NodePort pour accéder depuis l'extérieur
kubectl expose service postgres-ha-postgresql-ha \
  --type=NodePort \
  --target-port=5432 \
  --name=postgres-external \
  --namespace postgres-ha

# Obtenir l'URL d'accès
minikube service postgres-external --namespace postgres-ha --url
```

### Étape 10 : Surveillance et maintenance

Quelques commandes utiles pour surveiller votre déploiement :

```bash
# Vérifier les logs d'un pod PostgreSQL
kubectl logs -f postgres-ha-postgresql-ha-0 -n postgres-ha

# Vérifier l'état des répliques
kubectl get statefulsets -n postgres-ha

# Redémarrer un pod en cas de problème
kubectl delete pod postgres-ha-postgresql-ha-0 -n postgres-ha
```

### Étape 11 : Test de la haute disponibilité

Pour tester que la haute disponibilité fonctionne :

```bash
# Simuler une panne en supprimant un pod
kubectl delete pod postgres-ha-postgresql-ha-0 -n postgres-ha

# Vérifier que le pod est automatiquement recréé
kubectl get pods -n postgres-ha -w
```

**Explication :** Kubernetes va automatiquement recréer le pod supprimé, démontrant ainsi la haute disponibilité.

## 4. Commandes de gestion courantes

### Mise à jour de PostgreSQL

```bash
# Mettre à jour avec de nouveaux paramètres
helm upgrade postgres-ha bitnami/postgresql-ha \
  --namespace postgres-ha \
  --values postgres-ha-values.yaml

# Vérifier l'historique des déploiements
helm history postgres-ha -n postgres-ha
```

### Sauvegarde et restauration

```bash
# Créer une sauvegarde
kubectl exec -it postgres-ha-postgresql-ha-0 -n postgres-ha -- \
  pg_dump -U postgres ma_base_de_donnees > backup.sql

# Restaurer une sauvegarde
kubectl exec -i postgres-ha-postgresql-ha-0 -n postgres-ha -- \
  psql -U postgres ma_base_de_donnees < backup.sql
```

### Nettoyage

```bash
# Supprimer le déploiement PostgreSQL
helm uninstall postgres-ha -n postgres-ha

# Supprimer le namespace
kubectl delete namespace postgres-ha

# Arrêter minikube
minikube stop

# Supprimer complètement minikube
minikube delete
```

## 5. Résolution des problèmes courants

### Problème : Les pods ne démarrent pas

**Solution :**
```bash
# Vérifier les événements du namespace
kubectl get events -n postgres-ha --sort-by=.metadata.creationTimestamp

# Vérifier les logs d'un pod
kubectl logs postgres-ha-postgresql-ha-0 -n postgres-ha
```

### Problème : Pas assez de ressources

**Solution :**
```bash
# Augmenter les ressources de minikube
minikube stop
minikube start --memory=8192 --cpus=4
```

### Problème : Volume persistant non créé

**Solution :**
```bash
# Vérifier les storage classes disponibles
kubectl get storageclass

# Lister les volumes persistants
kubectl get pv
```

## 6. Conclusion

Ce guide vous a montré comment déployer PostgreSQL en haute disponibilité sur Kubernetes en utilisant Minikube, kubectl et Helm. Vous avez maintenant :

- Un cluster PostgreSQL avec 3 instances pour la haute disponibilité
- Un stockage persistant pour la durabilité des données
- Un système de répartition de charge avec pgpool
- Les outils pour surveiller et maintenir votre déploiement

PostgreSQL HA sur Kubernetes offre une solution robuste pour les applications nécessitant une base de données hautement disponible et scalable. N'hésitez pas à adapter la configuration selon vos besoins spécifiques.

## 7. Ressources supplémentaires

- **Documentation Kubernetes** : https://kubernetes.io/docs/
- **Documentation Helm** : https://helm.sh/docs/
- **Chart PostgreSQL HA de Bitnami** : https://github.com/bitnami/charts/tree/master/bitnami/postgresql-ha
- **Documentation PostgreSQL** : https://www.postgresql.org/docs/
