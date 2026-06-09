# Quelques questions


Q. Pourquoi utiliser des networks différents pour les db

Q. Quelle est la force et le principal intéret de Traefik en comparaison avec Nginx

Q. Monte un ou deux wordpress, modifie l'un d'entre eux, puis stop el et relance le, qu'observe-t-on ?

Q. Pour palier à ce souci ajoute des volumes aux Wordpress (et Maria)

Q. Qu'est-ce qu'un wildcard quand on parle de certificats TLS ? (ici on en utilisera pas)

Q. Ajoute une redirection HTTP ver HTTPS

```
- "--entrypoints.web.http.redirections.entryPoint.to=websecure"
- "--entrypoints.web.http.redirections.entryPoint.scheme=https"

```

Q. Comment fonctionne l'obtention de certificat Let's Encrypt avec Traefik ?

Q. Adapte les docker compose pour en comprendre le fonctionnement.

Q. Ajoute un conteneur de ton choix (NextCloud,Glpi...) en gardant la philosphie des labels et des networks

Q. Modifie le fichier C:\Windows\System32\drivers\etc\hosts pour simuler une resolution DNS vers tes wordpress et faire fonctionner le HTTPS (point vers 127.0.0.1)

exemple : 127.0.0.1 wp.poc2.local

/!\ le 127.0.0.1 est valable si tu montes docker sur un WSL, sinon il faudra pointer vers l'IP du lab (ou mieux générer un vrai certificat délivré par une CA)