# Support — Introduction à Docker, Docker Compose & Traefik

> **Version alpha** — Ce TP vise une première introduction à Docker, Traefik et à l'interaction entre conteneurs.
> Toute remarque est la bienvenue !

---

## Objectifs

- Installer Docker sur Debian via le dépôt officiel
- Sécuriser l'accès Docker pour un utilisateur non-root
- Comprendre les bases de Docker Compose
- Orchestrer une stack multi-conteneurs (Traefik + WordPress)
- Gérer la persistance des données avec les volumes
- Mettre en place la terminaison TLS avec Traefik

---

## Arborescence du projet

```
.
├── compose
│   ├── traefik
│   │   └── docker-compose.yml
│   ├── wordpress1
│   │   └── docker-compose.yml
│   └── wordpress2
│       └── docker-compose.yml
├── acme.json          ← certificats Let's Encrypt (chmod 600 !)
└── README.md
```

---

## 1. Installation de Docker sur Debian

### 1.1 Mise à jour et dépendances

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg
```

### 1.2 Ajout de la clé GPG et du dépôt Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 1.3 Installation du moteur Docker

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 1.4 Vérification de l'installation

```bash
docker --version
docker compose version
```

---

## 2. Accès Docker sans sudo

Par défaut, Docker nécessite les droits root. Pour éviter de préfixer chaque commande par `sudo`, on ajoute l'utilisateur courant au groupe `docker`.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

>  **Attention** : appartenir au groupe `docker` équivaut à avoir des droits root sur la machine. Ne le faites qu'en environnement de dev/TP.

---

## 3. Exercices pratiques

### 3.1 Lancer la stack Traefik / WordPress

Adapte les noms de domaine et les mots de passe dans les fichiers `docker-compose.yml` avant de démarrer.

>  **Pense à changer les mots de passe par défaut !**

```bash
# Depuis le dossier du service à lancer
docker compose up -d
```

### 3.2 Ajouter un nouveau conteneur

Choisis un service supplémentaire (Adminer, Nextcloud, Portainer,…) et crée son propre `docker-compose.yml` dans un nouveau sous-dossier de `compose/`. Expose-le via Traefik avec les bons labels.
On trouve facilement les docker compose sur les sites des éditeurs

### 3.3 Tester la persistance des données

1. Lance ta stack WordPress et crée ou modifie une page.
2. Stoppe et relance la stack :

```bash
docker compose down
docker compose up -d
```

3. Constate que les modifications ont disparu.
4. Ajoute un **volume** dans le `docker-compose.yml` pour persister le dossier `/var/www/html` (et la base de données).
5. Recommence l'étape 1 et vérifie que les données survivent au redémarrage.

---

## 4. Certificats TLS

### 4.1 Préparer le fichier de stockage ACME

Traefik stocke les certificats Let's Encrypt dans un fichier `acme.json`. Il **doit** avoir les permissions `600`, sinon Traefik refusera de démarrer.

```bash
touch acme.json && chmod 600 acme.json
```

### 4.2 Certificat auto-signé (environnement local)

Pour un environnement sans nom de domaine public, Traefik peut générer un certificat auto-signé à la volée. Ajoute les labels suivants au service concerné dans ton `docker-compose.yml` :

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.wp1.rule=Host(`site1.local`)"
  - "traefik.http.routers.wp1.entrypoints=websecure"
  # Certificat auto-signé généré par Traefik (pas de resolver nécessaire)
  - "traefik.http.routers.wp1.tls=true"
```

> Le navigateur affichera un avertissement de sécurité, ce qui est normal pour un cert auto-signé.

### 4.3 Certificat Let's Encrypt (environnement avec domaine public)

Si tu disposes d'un nom de domaine et d'une machine accessible depuis Internet, configure un **resolver ACME** dans Traefik pour obtenir un certificat valide automatiquement :

```yaml
# Dans la config statique de Traefik (traefik.yml ou args du conteneur)
certificatesResolvers:
  letsencrypt:
    acme:
      email: ton@email.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web
```

Puis dans les labels du service :

```yaml
labels:
  - "traefik.http.routers.wp1.tls.certresolver=letsencrypt"
```

---

## 5. Cheat Sheet

| Action | Commande |
|---|---|
| Lancer la stack en arrière-plan | `docker compose up -d` |
| Arrêter et supprimer les conteneurs | `docker compose down` |
| Lister les conteneurs actifs | `docker ps` |
| Voir les logs en temps réel | `docker compose logs -f` |
| Entrer dans un conteneur | `docker exec -it <nom_conteneur> bash` |
| Voir les volumes | `docker volume ls` |
| Inspecter un conteneur | `docker inspect <nom_conteneur>` |

---

## Ressources utiles

- [Documentation officielle Docker](https://docs.docker.com/)
- [Documentation Traefik](https://doc.traefik.io/traefik/)
- [Docker Hub](https://hub.docker.com/)
