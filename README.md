# TP Docker — Traefik & WordPress

> Introduction pratique à Docker, Docker Compose et au reverse proxy Traefik.
> L'objectif est de comprendre comment orchestrer plusieurs conteneurs qui communiquent entre eux, avec une terminaison TLS gérée centralement.

En complément le fichier Support.md offre une approche plus didactique ainsi que les composes Traefik et Wordpress "explained" pour expliquer chaques annotations.

Différentes approches sont possibles :

- Cours.md offre un peu (beaucoup) de lecture avec une petite manipulation de de conteneur
- Support.md est un parcours semi-guidée pour l'installation de docker 

/!\ Dans le cas d'un lab, il est évidement possible d'adapter les fqdn et d'utiliser un vrai domaine (ce qui simplifiera largement la partie sur les certificats)
Pour ce faire il suffit d'éditer les fichiers compose mis à disposition
---

## Architecture

```
Internet / Navigateur
        │
        ▼
   [ Traefik ]  ← reverse proxy, écoute sur :80 et :443
        │             lit les labels des conteneurs Docker
        │             gère les certificats TLS
        │
   traefik-net  ← réseau Docker partagé entre les stacks
        │
   ┌────┴────┐
   │         │
[ WP1 ]   [ WP2 ]   ← conteneurs WordPress (Apache + PHP)
   │         │
[ DB1 ]   [ DB2 ]   ← conteneurs MariaDB, isolés sur leurs réseaux internes
```

Chaque stack WordPress tourne de façon **indépendante** dans son propre dossier. Traefik est le seul point d'entrée : il reçoit tout le trafic et le redirige vers le bon WordPress selon le nom de domaine demandé.

---

## Prérequis

- Docker + Docker Compose installés ([guide d'installation](https://docs.docker.com/engine/install/debian/))
- Votre utilisateur dans le groupe `docker` (`sudo usermod -aG docker $USER`)
- Le réseau partagé créé **une seule fois** sur l'hôte :

```bash
docker network create traefik-net
```

---

## Démarrage rapide

### 1. Traefik

```bash
cd compose/traefik
docker compose up -d
```

Traefik est maintenant actif et surveille les nouveaux conteneurs. Son dashboard est accessible sur [http://localhost:8080](http://localhost:8080).

### 2. WordPress 1

```bash
cd compose/wordpress1
docker compose up -d
```

### 3. WordPress 2

```bash
cd compose/wordpress2
docker compose up -d
```

> Pour accéder aux sites en local, ajoute les entrées suivantes dans ton `/etc/hosts` :
> ```
> 127.0.0.1  site1.local
> 127.0.0.1  site2.local
> ```

---

## Structure des fichiers

```
.
├── compose
│   ├── traefik
│   │   └── docker-compose.yml   ← reverse proxy + gestion TLS
│   ├── wordpress1
│   │   └── docker-compose.yml   ← stack WP + MariaDB (site1.local)
│   └── wordpress2
│       └── docker-compose.yml   ← stack WP + MariaDB (site2.local)
├── letsencrypt
│   └── acme.json                ← certificats Let's Encrypt (chmod 600 !)
└── README.md
```

---

## Concepts abordés

| Concept | Où le voir dans ce TP |
|---|---|
| Image vs conteneur | `image: wordpress:latest` dans les composes |
| Variables d'environnement | Credentials MariaDB / WordPress |
| Réseaux Docker | `db1-net` (interne) vs `traefik-net` (partagé) |
| Volumes | Persistance de la BDD et des fichiers WordPress |
| Labels | Configuration dynamique de Traefik |
| Reverse proxy | Traefik route selon le `Host` HTTP |
| Terminaison TLS | Certificat auto-signé ou Let's Encrypt via Traefik |
| `depends_on` | Ordre de démarrage WordPress → MariaDB |

---

## TLS — deux modes

### Certificat auto-signé (dev local)

Aucune configuration supplémentaire dans Traefik. Dans le compose WordPress, il suffit d'activer TLS sans préciser de resolver :

```yaml
- "traefik.http.routers.wp1.tls=true"
```

Le navigateur affichera un avertissement de sécurité, ce qui est normal.

### Let's Encrypt (domaine public)

1. Décommenter les lignes ACME dans `compose/traefik/docker-compose.yml`
2. Renseigner une adresse email valide
3. Créer le fichier de stockage des certificats :

```bash
touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json
```

4. Dans le compose WordPress, s'assurer que le label pointe vers le bon resolver :

```yaml
- "traefik.http.routers.wp1.tls.certresolver=myresolver"
```

> Let's Encrypt applique un quota de **5 certificats par domaine par semaine**. En cas de tests répétés, utilise l'environnement de staging ACME pour éviter de le dépasser.

---

## Commandes utiles

```bash
# Démarrer une stack
docker compose up -d

# Arrêter une stack (les volumes sont conservés)
docker compose down

# Arrêter ET supprimer les volumes (reset complet)
docker compose down -v

# Voir les conteneurs actifs
docker ps

# Suivre les logs en temps réel
docker compose logs -f

# Entrer dans un conteneur
docker exec -it <nom_du_conteneur> bash

# Lister les réseaux Docker
docker network ls

# Inspecter un conteneur
docker inspect <nom_du_conteneur>
```

---

## Ressources

- [Documentation Docker](https://docs.docker.com/)
- [Documentation Traefik v2](https://doc.traefik.io/traefik/)
- [Docker Hub — WordPress](https://hub.docker.com/_/wordpress)
- [Docker Hub — MariaDB](https://hub.docker.com/_/mariadb)
