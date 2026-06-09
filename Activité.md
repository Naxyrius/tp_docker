# Activité — Mise en pratique

> Ce fichier est ton fil conducteur pour les exercices pratiques.
> L'objectif est de mettre les mains dans le cambouis : moins de guidage, plus d'autonomie.
> Le `Support.md` et les fichiers `compose-explained.yml` sont là si tu bloques.

---

## Activité 1 — Compléter la stack WordPress 2

Le fichier `compose/wordpress2/docker-compose.yml` est vide. À toi de le compléter pour avoir un second WordPress fonctionnel, accessible sur `site2.local`, qui tourne en parallèle du premier.

### Contraintes

- Le second WordPress doit être **indépendant** du premier (réseaux, volumes et credentials séparés)
- Il doit être routé par Traefik, comme `wordpress1`
- **Attention aux conflits** : tous les noms qui doivent être uniques sur l'hôte (réseaux internes, noms de routeurs Traefik, noms de volumes)
- La base de données ne doit pas être accessible depuis l'extérieur

### Critères de validation

- [ ] `docker compose up -d` dans `wordpress2/` se lance sans erreur
- [ ] `site2.local` est accessible dans le navigateur
- [ ] `site1.local` fonctionne toujours (pas de régression)
- [ ] Le dashboard Traefik affiche bien deux routeurs distincts
- [ ] Les données survivent à un `docker compose down` / `up`

### Coup de pouce

> Si tu es bloqué sur le démarrage, commence par relire le `compose-explained.yml` de `wordpress1` en te demandant pour chaque ligne : *"qu'est-ce qui doit changer pour une seconde instance ?"*

---

## Activité 2 — Ajouter un service de ton choix

Tu as maintenant une stack Traefik + deux WordPress opérationnels. L'objectif est d'y greffer un **service supplémentaire** de ton choix, exposé via Traefik.

### Quelques idées

| Service | Utilité |
|---|---|
| **Adminer** | Interface web légère pour gérer une BDD |
| **Portainer** | Dashboard de gestion de conteneurs Docker |
| **Whoami** | Micro-service qui affiche les infos de la requête reçue (idéal pour tester Traefik) |
| **Nextcloud** | Stockage de fichiers auto-hébergé |
| **Uptime Kuma** | Monitoring de disponibilité |

> Les images officielles et leurs exemples de compose se trouvent sur [Docker Hub](https://hub.docker.com). Les éditeurs publient souvent des exemples directement dans leur documentation.

### Ce qu'on attend

- Un `docker-compose.yml` dans un nouveau sous-dossier de `compose/`
- Le service exposé via Traefik sur un nom de domaine local de ton choix
- Les données persistées si le service en a besoin

### C'est volontairement ouvert

Il n'y a pas de marche à suivre ici. Explore, teste, casse des choses et répare-les.
C'est comme ça qu'on apprend Docker.

---

> **Rappel** : en cas de panne, le [guide de debug](../Support.md#7-guide-de-debug) dans `Support.md` suit une logique éprouvée. Commence toujours par `docker compose logs -f` avant de chercher ailleurs.
