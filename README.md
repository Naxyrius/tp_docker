Ce tp est une version "alpha" visant une première introduction à Docker, traefik et à l'intéraction des conteneurs.
Toute remarque est bonne à prendre ! 

# TP Introduction à Docker (et docker compose)

## Objectifs : 
* Installer Docker sur Debian (avec le dépôt officiel)
* Sécuriser l'accès Docker pour l'utilisateur
* Orchestrer une stack multi-conteneurs

## Installation de Docker 

### MaJ et installation des depandances

```
sudo apt update
sudo apt install ca-certificates curl gnupg

```

### Ajout de la clé GPG et du repo docker

```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

### Installation du moteur 

```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```

## Ajout du user dans le groupe docker : 

```
sudo usermod -aG docker $USER
newgrp docker

```
## Certificat

N'oublie pas de créer le fichier acme.json avec les bons droits (touch acme.json && chmod 600 acme.json) avant de lancer le conteneur.

```
touch acme.json && chmod 600 acme.json
```


# Cheat Sheet 

Lancer un conteneur depuis un compose :

```
docker compose up -d

```

Tout stopper  :

```
docker compose down

```

Afficher les conteneurs : 

```
docker ps
```


Afficher les logs :

```
docker compose logs -f

```

Rentrer dans le conteneur : 

```
docker exec -it 
```

# Via Docker Compose 

Adapte les noms et monte la stack Traefik / Wordpress
/!\ Modifie les mdp :D

Ajoute un nouveau containeurs du service de ton choix et adapte son docker compose

Modifie une page d'un wordpress, puis fait un down/up.

-> Tu dois avoir perdu des petits, on ajoute donc un volume pour la persistance des données



# Certificat
En fonction de l'environnement, ajoute un certificat let's encrypt au niveau de Traefik pour gérer la terminaison TLS

Pour du certificat auto-signé par traefik on ajoute ça au compose de traefik

```
labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wp1.rule=Host(`site1.local`)"
      - "traefik.http.routers.wp1.entrypoints=websecure"
      # On dit à Traefik de générer un certificat local d'office, sans resolver :
      - "traefik.http.routers.wp1.tls=true"

``` 