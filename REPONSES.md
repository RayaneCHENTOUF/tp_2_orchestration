# TP Jour 2 — Réponses aux questions de compréhension

## 1. Différence entre une image Docker et un conteneur Docker

Une **image Docker** peut être vue comme un modèle qui contient tout ce dont une application a besoin pour fonctionner : le système de base, les dépendances, le code de l'application et la commande à lancer. Une image est en lecture seule et peut être utilisée pour créer plusieurs conteneurs.

Un **conteneur** est une instance de cette image qui est réellement lancée. Il possède son propre environnement isolé et peut être démarré, arrêté ou supprimé.

Par exemple, une même image `mon-service:1.0` peut servir à lancer plusieurs conteneurs, comme les 3 puis les 5 pods utilisés dans le Deployment.

## 2. Pourquoi les données sont perdues si l'on supprime le volume, mais pas le conteneur ?

Un conteneur est généralement considéré comme temporaire. Les données écrites directement à l'intérieur du conteneur sont donc perdues lorsque celui-ci est supprimé.

Dans notre cas, PostgreSQL stocke ses données dans `/var/lib/postgresql/data`. Ce dossier est associé au volume Docker `donnees_db`.

Le volume est indépendant du conteneur. C'est pour cela que lorsque l'on utilise `docker compose down`, les conteneurs sont supprimés mais le volume reste présent. En redémarrant les services, PostgreSQL peut donc retrouver les données précédentes.

En revanche, avec `docker compose down -v`, le volume est également supprimé. La base repart alors de zéro et le fichier `init.sql` est exécuté à nouveau sur cette nouvelle base.

## 3. Un Deployment `replicas: 3` perd un pod : que fait Kubernetes ? Pourquoi parle-t-on d'approche « déclarative » ?

Si un Deployment est configuré avec `replicas: 3` et qu'un des pods est supprimé, Kubernetes remarque qu'il ne reste plus que 2 pods alors que 3 sont demandés.

Le Deployment, grâce au ReplicaSet qu'il gère, va donc automatiquement créer un nouveau pod pour revenir à 3.

C'est ce que nous avons pu observer pendant le TP : après la suppression du pod `api-deployment-...-rjpbl`, Kubernetes en a automatiquement créé un nouveau, `api-deployment-...-7bmk7`.

On parle d'approche **déclarative** car on indique simplement à Kubernetes l'état que l'on souhaite obtenir, par exemple `replicas: 3`, plutôt que de lui donner toutes les étapes à effectuer.

Kubernetes vérifie ensuite régulièrement la différence entre l'état souhaité et l'état réel, puis effectue les changements nécessaires pour revenir à l'état demandé.

## 4. Quel est le rôle du Service par rapport aux pods ?

Les pods peuvent être supprimés ou recréés à tout moment. Leur adresse IP peut donc changer, ce qui rendrait compliqué l'accès direct à un pod précis.

Le **Service** permet justement d'avoir un point d'accès stable pour communiquer avec les pods. Dans notre cas, le Service est accessible avec le nom `api-service` et possède une adresse IP de cluster stable. Il est également exposé avec un `NodePort`.

Le Service utilise les labels, par exemple `app: mon-service`, pour identifier les pods concernés. Il peut ensuite envoyer les requêtes vers les différents pods disponibles.

Ainsi, les utilisateurs ou les autres applications n'ont pas besoin de connaître directement les adresses IP des pods.

## 5. Deux limites de Docker seul, sans Kubernetes

Docker permet déjà de lancer et gérer des conteneurs, mais il présente certaines limites lorsqu'on veut gérer une application avec plusieurs instances.

- **Pas de véritable auto-réparation :** si un conteneur tombe, certaines options comme `restart: always` permettent de le redémarrer, mais cela reste limité au même hôte. Si la machine entière tombe, Docker ne peut pas automatiquement déplacer le conteneur sur une autre machine.

- **Mise à l'échelle limitée :** avec Docker Compose, les conteneurs sont généralement lancés sur un seul hôte. Lancer plusieurs instances peut notamment créer des problèmes de ports, comme avec `8000:8000`. Docker seul ne fournit pas non plus un système complet de répartition du trafic entre plusieurs machines.

On peut également citer d'autres limites, comme l'absence de gestion avancée des déploiements progressifs, des rollbacks ou de l'ordonnancement automatique des conteneurs sur plusieurs machines en fonction des ressources disponibles.
