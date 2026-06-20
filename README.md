# BookStack Helm chart

Chart Helm minimal pour déployer BookStack et MariaDB dans Kubernetes.

Le chart génère actuellement :

- un `Deployment` et un `Service` pour BookStack ;
- un `StatefulSet` et un Service headless pour MariaDB ;
- un PVC pour les données BookStack et un PVC pour les données MariaDB.

## Installation

Depuis la racine du dépôt :

```bash
helm lint ./bookstack-chart
helm upgrade --install bookstack ./bookstack-chart \
  --namespace bookstack \
  --create-namespace
```

Pour afficher les manifestes sans les appliquer :

```bash
helm template bookstack ./bookstack-chart --namespace bookstack
```

Le Service BookStack est de type `ClusterIP`. En développement local, on peut y accéder ainsi :

```bash
kubectl -n bookstack port-forward service/bookstack 6875:80
```

L'application est alors accessible sur `http://localhost:6875`.

## Configuration

Les ressources sont des listes dans `bookstack-chart/values.yaml`. On peut donc ajouter plusieurs éléments du même type :

```yaml
services:
  - name: bookstack
    enabled: true
    # ...
  - name: another-service
    enabled: true
    # ...
```

Le champ `enabled: false` empêche le rendu de l'élément concerné. Les références doivent correspondre aux noms déclarés :

- `services[].selectorName` doit correspondre au `name` du Deployment ou StatefulSet ciblé ;
- `volumes[].persistentVolumeClaim.claimName` doit correspondre à `pvcs[].name` ;
- `DB_HOST` doit correspondre au nom du Service MariaDB (`mariadb` par défaut).

## Clé et mots de passe

Avant un déploiement autre que local, adapte les valeurs suivantes dans `values.yaml` :

- `deployments[0].env.APP_KEY` : clé Laravel propre à l'instance BookStack. Elle ne doit pas être changée après le premier démarrage, sinon les données chiffrées existantes ne pourront plus être lues. Une clé peut être générée avec `openssl rand -base64 32`, puis préfixée par `base64:`.
- `deployments[0].env.APP_URL` : URL publique exacte de BookStack. Pour un accès via Ingress, utilise par exemple `https://bookstack.example.com`.
- `deployments[0].env.DB_PASSWORD` et `statefulsets[0].env.MARIADB_PASSWORD` : ces deux valeurs doivent être identiques.
- `statefulsets[0].env.MARIADB_ROOT_PASSWORD` : mot de passe administrateur MariaDB, à définir avec une valeur forte.

Les mots de passe présents dans le fichier sont des exemples de développement. Ne les utilise pas en production et ne versionne pas de vraies clés ou mots de passe dans Git. Pour un usage réel, déplace-les vers des `Secret` Kubernetes ou une solution de gestion de secrets.

## Stockage et exposition

Les PVC utilisent `ReadWriteOnce` : ton cluster doit disposer d'une `StorageClass` par défaut ou d'un provisioner de stockage. Le StatefulSet MariaDB est configuré pour une seule réplique ; son PVC statique n'est pas adapté à une montée en charge multi-réplicas. Pour cela, il faudrait utiliser `volumeClaimTemplates`.

Pour exposer BookStack hors du cluster, ajoute un Ingress, ou adapte le Service en `LoadBalancer` / `NodePort`, puis mets à jour `APP_URL`.
