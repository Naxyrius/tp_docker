# Cours — Introduction à Docker, Docker Compose & Traefik

> Document de support didactique, à lire en complément du `README.md` et des fichiers `compose-explained.yml`.
> Guide d'installation de docker dans le fichier Support
---

## Table des matières

1. [Concepts fondamentaux](#1-concepts-fondamentaux)
2. [Les réseaux Docker](#2-les-réseaux-docker)
3. [Les volumes](#3-les-volumes)
4. [Traefik — comment ça marche vraiment](#4-traefik--comment-ça-marche-vraiment)
5. [Les labels Traefik décryptés](#5-les-labels-traefik-décryptés)
6. [Pièges classiques](#6-pièges-classiques)
7. [Guide de debug](#7-guide-de-debug)
8. [Exercice volumes — guide pas à pas](#8-exercice-volumes--guide-pas-à-pas)

---

## 1. Concepts fondamentaux

### Image vs Conteneur vs Service

Une **image** est un modèle figé, en lecture seule. Elle décrit un système de fichiers et une commande de démarrage. On ne l'exécute pas directement.

Un **conteneur** est une instance en cours d'exécution d'une image. C'est l'image + une couche d'écriture + un processus actif. On peut en lancer plusieurs à partir de la même image.

Un **service** est le nom qu'on donne à un conteneur dans un fichier Docker Compose. Il regroupe la définition de l'image, des variables d'environnement, des réseaux, des volumes, etc.

```
Image WordPress  →  Conteneur wp1  (site1.local)
                 →  Conteneur wp2  (site2.local)
```

> Analogie : l'image est le moule, le conteneur est la pièce coulée. On peut faire autant de pièces identiques qu'on veut avec le même moule.

### Ce qu'il se passe au `docker compose up`

1. Docker télécharge les images manquantes depuis Docker Hub
2. Il crée les réseaux déclarés dans le compose (sauf les `external`)
3. Il crée les volumes déclarés
4. Il démarre les conteneurs dans l'ordre (en tenant compte des `depends_on`)
5. Il attache chaque conteneur aux réseaux déclarés

---

## 2. Les réseaux Docker

Par défaut, les conteneurs sont isolés. Pour qu'ils puissent se parler, ils doivent être sur le même réseau Docker.

### Résolution DNS automatique

Docker intègre un serveur DNS interne. Sur un réseau partagé, chaque conteneur est joignable par son **nom de service**. C'est pourquoi on peut écrire dans le compose WordPress :

```yaml
WORDPRESS_DB_HOST: db:3306
```

`db` n'est pas une IP : c'est le nom du service MariaDB. Docker le résout automatiquement en adresse IP interne. Si on renommait le service `mariadb`, il faudrait écrire `mariadb:3306`.

### Pourquoi plusieurs réseaux dans ce TP ?

```
[ Traefik ] ──── traefik-net ──── [ WordPress ]
                                       │
                                   db1-net
                                       │
                                  [ MariaDB ]
```

On utilise deux réseaux par stack WordPress pour des raisons de sécurité et d'isolation :

- `traefik-net` est partagé entre toutes les stacks et Traefik. Tout ce qui y est connecté peut potentiellement recevoir du trafic depuis l'extérieur.
- `db1-net` est **privé** à la stack wordpress1. MariaDB n'y est pas exposée à Traefik ni aux autres stacks. Si un autre conteneur est compromis, il ne peut pas atteindre la base de données directement.

WordPress a besoin des deux réseaux : il doit parler à MariaDB **et** être accessible via Traefik. C'est le seul conteneur de la stack avec un "pied dans les deux mondes".

### Réseau external vs réseau géré par Compose

```yaml
networks:
  db1-net:           # Créé et géré par ce compose (supprimé au docker compose down)
  traefik-net:
    external: true   # Doit exister avant le docker compose up
```

Le réseau `traefik-net` est `external` parce qu'il est partagé entre plusieurs stacks indépendantes. Si Compose le gérait, il serait détruit au `docker compose down` de la stack Traefik, cassant toutes les autres stacks.

---

## 3. Les volumes

Sans volume, toutes les données écrites dans un conteneur disparaissent à sa destruction. C'est le comportement par défaut, et c'est ce que l'exercice 3.3 vous fait constater.

### Deux types de montage

**Bind mount** — on monte un dossier de l'hôte dans le conteneur :

```yaml
volumes:
  - ./letsencrypt:/letsencrypt   # dossier_hôte:dossier_conteneur
  - /var/run/docker.sock:/var/run/docker.sock
```

Le chemin de gauche est sur la machine hôte. C'est ce qu'on utilise pour `acme.json` et le socket Docker parce qu'on veut accéder à des fichiers spécifiques de l'hôte.

**Volume nommé** — Docker gère le stockage, on lui donne juste un nom :

```yaml
volumes:
  - wp1_data:/var/www/html   # nom_volume:dossier_conteneur

# En bas du compose, on déclare le volume
volumes:
  wp1_data:
```

C'est la méthode recommandée pour les données applicatives (BDD, fichiers WordPress). Docker stocke les données dans `/var/lib/docker/volumes/` et les gère indépendamment du cycle de vie des conteneurs.

### `docker compose down` vs `docker compose down -v`

```bash
docker compose down        # Supprime les conteneurs et réseaux, conserve les volumes
docker compose down -v     # Supprime aussi les volumes → reset complet
```

>  `down -v` est destructif. Ne jamais l'utiliser en production sans sauvegarde.

---

## 4. Traefik — comment ça marche vraiment

Traefik est un **reverse proxy dynamique**. Contrairement à Nginx où on édite des fichiers de configuration, Traefik se configure automatiquement en lisant les labels des conteneurs Docker.

### Le flux d'une requête

```
Navigateur → site1.local
    │
    ▼
[ Traefik :443 ]
    │
    │  Lit le header HTTP "Host: site1.local"
    │  Cherche un routeur dont la rule matche
    │  Trouve : traefik.http.routers.wp1.rule=Host(`site1.local`)
    │
    ▼
[ WordPress — port 80 interne ]
    │
    ▼
Réponse HTTP → Traefik → Navigateur
```

Traefik gère la terminaison TLS : la connexion entre le navigateur et Traefik est chiffrée (HTTPS), mais la connexion entre Traefik et WordPress peut rester en HTTP simple sur le réseau interne. C'est le fonctionnement standard d'un reverse proxy.

### Comment Traefik découvre les conteneurs

Grâce à la ligne `--providers.docker=true` et au montage du socket Docker :

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

Traefik écoute les événements Docker en temps réel. Dès qu'un conteneur démarre avec `traefik.enable=true`, Traefik met à jour sa configuration à chaud, sans redémarrage.

### Le dashboard Traefik

Accessible sur `http://localhost:8080` (activé par `--api.insecure=true`), il permet de visualiser en temps réel :

- Les **routers** : les règles de routage actives
- Les **services** : les conteneurs cibles
- Les **middlewares** : les transformations appliquées aux requêtes
- Les **certificats** TLS en cours

C'est le premier endroit à consulter quand une route ne fonctionne pas.

---

## 5. Les labels Traefik décryptés

```yaml
labels:
  - "traefik.enable=true"
  #   └─ Active Traefik pour ce conteneur. Sans ça, il est ignoré.

  - "traefik.http.routers.wp1.rule=Host(`site1.local`)"
  #   └─ Crée un routeur nommé "wp1"
  #      qui répond aux requêtes dont le Host est "site1.local"

  - "traefik.http.routers.wp1.entrypoints=websecure"
  #   └─ Ce routeur écoute sur l'entrypoint "websecure" (port 443)
  #      Le nom "websecure" doit correspondre à ce qui est déclaré dans Traefik :
  #      --entrypoints.websecure.address=:443

  - "traefik.http.routers.wp1.tls.certresolver=myresolver"
  #   └─ TLS activé, certificat géré par le resolver "myresolver"
  #      Pour un cert auto-signé local, remplacer par :
  #      "traefik.http.routers.wp1.tls=true"

  - "traefik.http.services.wp1.loadbalancer.server.port=80"
  #   └─ Traefik doit forwarder le trafic vers le port 80 du conteneur
  #      (WordPress/Apache écoute sur 80 en interne)
```

### Le lien entre les noms

Le nom `wp1` dans les labels doit être **unique** par stack. Si vous avez deux WordPress avec le même nom de routeur, Traefik ne saura pas lequel choisir. Dans `wordpress2`, utilisez `wp2` partout.

Le nom `myresolver` dans les labels **doit correspondre exactement** au nom déclaré dans la config Traefik :
```
--certificatesresolvers.myresolver.acme.tlschallenge=true
                        ^^^^^^^^^^
                        même nom ici
```

---

## 6. Pièges classiques

### Le réseau `traefik-net` n'existe pas

**Symptôme** : `docker compose up` échoue avec une erreur `network traefik-net declared as external, but could not be found`.

**Solution** : créer le réseau avant de lancer quoi que ce soit :
```bash
docker network create traefik-net
```
À ne faire qu'**une seule fois** sur l'hôte.

---

### Les mots de passe MariaDB ne correspondent pas

**Symptôme** : WordPress démarre mais affiche "Error establishing a database connection".

**Cause** : `MYSQL_PASSWORD` dans le service `db` et `WORDPRESS_DB_PASSWORD` dans le service `wordpress` ont des valeurs différentes.

**Solution** : vérifier que les quatre variables sont cohérentes entre les deux services :

```yaml
# Service db
MYSQL_DATABASE: wp1
MYSQL_USER: wpuser1
MYSQL_PASSWORD: monpassword   ← doit être identique

# Service wordpress
WORDPRESS_DB_NAME: wp1        ← même valeur
WORDPRESS_DB_USER: wpuser1    ← même valeur
WORDPRESS_DB_PASSWORD: monpassword  ← même valeur
```

---

### `depends_on` ne suffit pas toujours

**Symptôme** : WordPress crash au démarrage avec une erreur de connexion BDD, même avec `depends_on: db`.

**Cause** : `depends_on` attend que le conteneur `db` soit **démarré**, pas que MariaDB soit **prête à accepter des connexions**. L'initialisation de la BDD au premier lancement prend quelques secondes.

**Solution** : WordPress détecte l'échec et réessaie automatiquement. En général, laisser 10-15 secondes et rafraîchir. Si le problème persiste, vérifier les mots de passe en premier.

Pour une solution robuste (hors scope de ce TP) : utiliser `healthcheck` sur le service `db`.

---

### `acme.json` a les mauvaises permissions

**Symptôme** : Traefik démarre mais les certificats Let's Encrypt ne sont jamais obtenus. Les logs indiquent `acme.json` has wrong permissions.

**Solution** :
```bash
chmod 600 letsencrypt/acme.json
```
Traefik refuse de lire ou d'écrire dans ce fichier s'il n'est pas en `600`, pour éviter que les certificats soient lisibles par d'autres processus.

---

### Traefik ne voit pas un conteneur

**Symptôme** : le site est inaccessible, le routeur n'apparaît pas dans le dashboard Traefik.

**Causes possibles** :
- `traefik.enable=true` manquant dans les labels
- Le conteneur n'est pas sur le réseau `traefik-net`
- Le nom de l'entrypoint dans les labels (`websecure`) ne correspond pas à celui déclaré dans Traefik

**Vérification rapide** :
```bash
# Voir les réseaux d'un conteneur
docker inspect <nom_conteneur> | grep -A 20 Networks

# Voir tous les labels d'un conteneur
docker inspect <nom_conteneur> | grep -A 30 Labels
```

---

###  Incompatibilité API Docker / Traefik

**Symptôme** : Traefik démarre mais ne détecte aucun conteneur, même avec des labels corrects. Les logs affichent une erreur du type :

```
Error while fetching server version: client version X.XX is too new. Maximum supported API version is 1.XX
```

**Cause** : le provider Docker de Traefik utilise l'API Docker pour lire les métadonnées des conteneurs. Les versions récentes de Docker Engine exposent des versions d'API élevées (1.44+), et certaines versions de Traefik 2.x (notamment 2.10 et antérieures) peuvent rencontrer des problèmes de compatibilité avec ces nouvelles versions d'API.

**Solution** : forcer la version d'API utilisée par Traefik via la variable d'environnement `DOCKER_API_VERSION` :

```yaml
services:
  traefik:
    image: traefik:v2.11
    environment:
      - DOCKER_API_VERSION=1.40
    ...
```

La version `1.40` est une valeur sûre et compatible avec l'ensemble des fonctionnalités utilisées par Traefik 2.x. En cas de doute, consulter la version d'API exposée par votre daemon Docker :

```bash
docker version --format '{{.Server.APIVersion}}'
```

> Cette contrainte disparaît avec Traefik v3, qui gère mieux la négociation de version d'API.

---

## 7. Guide de debug

Quand quelque chose ne fonctionne pas, suivre cet ordre :

### Étape 1 — Les conteneurs tournent-ils ?

```bash
docker ps
```

Si un conteneur est absent ou en statut `Exited`, regarder pourquoi :

```bash
docker compose logs <nom_service>
# ou pour tout voir
docker compose logs -f
```

### Étape 2 — Les réseaux sont-ils corrects ?

```bash
docker network ls
# Vérifier que traefik-net existe

docker network inspect traefik-net
# Vérifier que Traefik ET WordPress sont listés dans "Containers"
```

### Étape 3 — Traefik voit-il le conteneur ?

Ouvrir le dashboard sur `http://localhost:8080` et vérifier :
- L'onglet **HTTP → Routers** : le routeur `wp1` est-il présent ? Son statut est-il `Enabled` ?
- L'onglet **HTTP → Services** : le service `wp1` pointe-t-il vers la bonne IP/port ?

### Étape 4 — La résolution DNS locale fonctionne-t-elle ?

```bash
ping site1.local
# Doit répondre sur 127.0.0.1

curl -H "Host: site1.local" http://localhost
# Teste le routage HTTP sans passer par le DNS
```

### Étape 5 — WordPress peut-il joindre MariaDB ?

```bash
docker exec -it <nom_conteneur_wordpress> bash

# Dans le conteneur :
apt update && apt install -y mariadb-client
mysql -h db -u wpuser1 -p
# Entrer le mot de passe → si ça se connecte, la BDD est OK
```

---

## 8. Exercice volumes — guide pas à pas

L'objectif est d'ajouter la persistance des données à la stack `wordpress1`.

### Ce qu'on veut persister

- Les **fichiers WordPress** dans `/var/www/html` (thèmes, plugins, médias uploadés)
- La **base de données MariaDB** dans `/var/lib/mysql`

### Modifier le `docker-compose.yml`

Voici la structure à atteindre. Les lignes à ajouter sont indiquées :

```yaml
services:
  db:
    image: mariadb:10.11
    environment:
      MYSQL_DATABASE: wp1
      MYSQL_USER: wpuser1
      MYSQL_PASSWORD: password1
      MYSQL_ROOT_PASSWORD: rootpassword1
    volumes:
      - db1_data:/var/lib/mysql     # ← AJOUTER
    networks:
      - db1-net

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser1
      WORDPRESS_DB_PASSWORD: password1
      WORDPRESS_DB_NAME: wp1
    volumes:
      - wp1_data:/var/www/html      # ← AJOUTER
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wp1.rule=Host(`site1.local`)"
      - "traefik.http.routers.wp1.entrypoints=websecure"
      - "traefik.http.routers.wp1.tls=true"
      - "traefik.http.services.wp1.loadbalancer.server.port=80"
    networks:
      - db1-net
      - traefik-net

# ← AJOUTER ce bloc en bas du fichier
volumes:
  wp1_data:
  db1_data:

networks:
  db1-net:
  traefik-net:
    external: true
```

### Tester la persistance

```bash
# 1. Lancer la stack
docker compose up -d

# 2. Aller sur site1.local, créer une page ou modifier du contenu

# 3. Détruire les conteneurs (sans -v : les volumes sont conservés)
docker compose down

# 4. Relancer
docker compose up -d

# 5. Retourner sur site1.local → les données doivent être là
```

### Vérifier que les volumes existent

```bash
docker volume ls
# Vous devez voir : wordpress1_wp1_data et wordpress1_db1_data
# (Compose préfixe les volumes avec le nom du dossier)

docker volume inspect wordpress1_wp1_data
# Affiche le chemin réel sur l'hôte sous "Mountpoint"
```
