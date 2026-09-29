# TP Jour 2 — Réponses aux questions de compréhension

## 1. Différence entre une image Docker et un conteneur Docker

Une **image** est un modèle immuable, en lecture seule, construit à partir d'un Dockerfile : elle
empile des couches (système de base, dépendances, code de l'application) et décrit la commande
de démarrage. Un **conteneur** est une instance en cours d'exécution de cette image : Docker lui
ajoute une couche inscriptible, un processus isolé, un réseau et un cycle de vie propre
(créé, démarré, arrêté, supprimé). Une même image (`mon-service:1.0`) peut lancer plusieurs
conteneurs, comme les 3 puis 5 pods du Deployment.

## 2. Pourquoi les données sont perdues si l'on supprime le volume, mais pas le conteneur ?

Le conteneur est éphémère : tout ce qu'il écrit dans sa couche inscriptible disparaît avec lui.
Ici, PostgreSQL écrit dans `/var/lib/postgresql/data`, qui est monté sur le volume nommé
`donnees_db`. Ce volume est géré par Docker **indépendamment** du cycle de vie des conteneurs :
`docker compose down` supprime les conteneurs mais conserve le volume, donc au redémarrage la
base retrouve ses fichiers. `docker compose down -v` supprime aussi le volume : les données
disparaissent, et au démarrage suivant `init.sql` est rejoué sur une base vide.

## 3. Un Deployment `replicas: 3` perd un pod : que fait Kubernetes ? Pourquoi « déclaratif » ?

Le contrôleur (ReplicaSet géré par le Deployment) détecte que l'état observé (2 pods) ne
correspond plus à l'état désiré (3 pods) et crée immédiatement un nouveau pod pour revenir à 3.
Observé pendant le TP : le pod `api-deployment-...-k5hvq` supprimé a été remplacé aussitôt par
`api-deployment-...-nxdkw`.

On parle d'approche **déclarative** car on ne décrit pas les actions à effectuer (« lance un
conteneur », « relance-le s'il meurt »), mais l'**état final souhaité** dans un manifest YAML.
Kubernetes exécute en permanence une boucle de réconciliation qui compare l'état réel à l'état
déclaré et corrige les écarts, sans intervention humaine.

## 4. Rôle du Service par rapport aux pods

Les pods sont éphémères : à chaque recréation ou mise à l'échelle, ils obtiennent une nouvelle
adresse IP. Le **Service** fournit un point d'accès stable (nom DNS `api-service` et IP de
cluster fixe, ici exposée en plus sur un port du nœud via `NodePort`). Il sélectionne les pods
grâce à leurs labels (`app: mon-service`), met à jour automatiquement la liste des pods
disponibles et répartit le trafic entre eux. Les clients n'ont donc jamais besoin de connaître les
IP des pods.

## 5. Deux limites de Docker seul (sans Kubernetes)

- **Pas d'auto-réparation** : si un conteneur ou la machine tombe, rien ne le recrée ailleurs
  (`restart: always` ne relance que sur le même hôte, et ne gère pas la perte de l'hôte).
- **Pas de mise à l'échelle ni de répartition de charge natives** : Docker Compose tourne sur un
  seul hôte ; lancer plusieurs réplicas pose des conflits de ports (`8000:8000`) et il n'y a pas
  de load balancer intégré ni de découverte de service multi-machines.
- (Autres) Pas de déploiements progressifs (rolling update / rollback) ni d'ordonnancement sur
  plusieurs nœuds en fonction des ressources disponibles.
