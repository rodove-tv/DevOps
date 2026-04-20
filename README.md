Exercice 1 — Premier contact avec Docker

1.1

```
docker pull nginx:alpine
```

1.2

```
docker run -d --name mon-nginx -p 8080:80 nginx:alpine
```

1.3

La commande qui permet de lister uniquement les conteneurs en cours d'exécution est :

```
docker ps
```

1.4

En ouvrant http://localhost:8080, on voit la page d'accueil par défaut de Nginx avec le message "Welcome to nginx!". Cela confirme que le serveur web fonctionne correctement à l'intérieur du conteneur.

1.5

```
docker logs mon-nginx
```

1.6

```
docker stop mon-nginx
```

Pour lister tous les conteneurs (y compris arrêtés) :

```
docker ps -a
```

Différence : docker ps n'affiche que les processus actifs. docker ps -a (all) affiche l'historique de tous les conteneurs présents sur la machine, qu'ils soient allumés ou éteints.

1.7

```
docker rm mon-nginx
```

1.8

L'option --rm permet de supprimer automatiquement le conteneur dès qu'il est arrêté :

```
docker run -d --name mon-nginx -p 8080:80 --rm nginx:alpine
```

---

Exercice 2 — Construire sa première image avec un Dockerfile

2.1

Contenu du fichier index.html :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Ma première image Docker</title>
</head>
<body>
    <h1>Bonjour, je m'appelle [Votre Prénom]</h1>
</body>
</html>
```

2.2

Contenu du Dockerfile :

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

2.3

```
docker build -t mon-site:v1 .\exo2\
```

2.4

```
docker run -d -p 9090:80 --rm --name mon-site-v1 mon-site:v1
```

2.5

La taille de mon-site:v1 est pratiquement identique à celle de nginx:alpine (environ 5 Mo) car le fichier HTML ajouté est minuscule et l'image de base est la même.

2.6

```
docker history mon-site:v1
```

On remarque qu'une seule nouvelle couche (layer) a été ajoutée pour l'instruction COPY par rapport à l'image Nginx de base.

2.7

Lors de la reconstruction en v2 après modification :

Étape rechargée depuis le cache : L'étape FROM (l'image de base).

Étape réexécutée : L'étape COPY, car le contenu du fichier index.html a changé, ce qui invalide le cache pour cette ligne.

2.8

```
docker rmi mon-site:v1
```

---

Exercice 3 — Volumes et persistance des données

3.1

Après avoir redémarré un conteneur Alpine, le fichier /data/test.txt créé précédemment a disparu. C'est parce que les conteneurs sont éphémères : toute modification du système de fichiers interne est perdue à la suppression du conteneur si elle n'est pas liée à un volume.

3.2

```
docker run -d -p 8080:80 --rm --name nginx-mount -v "$(pwd)/exo3/html:/usr/share/nginx/html" nginx:alpine
```

On voit les modifications en direct sur le navigateur sans redémarrer le conteneur car le dossier est "monté" (bind mount) : le conteneur accède directement aux fichiers de l'hôte.

3.3 à 3.5

Création et utilisation d'un volume nommé :

```
docker volume create mes-donnees
docker run -it --rm -v mes-donnees:/data alpine sh
# (Dans le shell : echo "je survis" > /data/persistant.txt)
```

Même après l'arrêt et le lancement d'un nouveau conteneur, les données survivent. Cela démontre que les volumes nommés persistent les données indépendamment du cycle de vie des conteneurs.

3.6

```
docker volume inspect mes-donnees
```

Docker stocke cela dans un dossier géré par le démon (sous Linux : /var/lib/docker/volumes/).

3.7

```
docker volume rm mes-donnees
```

Il faut s'assurer qu'aucun conteneur n'est rattaché à ce volume avant de tenter sa suppression.

---

Exercice 4 — Réseaux Docker

4.1

Les trois réseaux par défaut créés par Docker sont : bridge, host, et none.

4.2

```
docker network create mon-reseau
```

4.3

```
docker run -d --name serveur-web --network mon-reseau nginx:alpine
```

4.4

Depuis un conteneur client sur le même réseau :

```
docker run -it --rm --network mon-reseau alpine
# wget -qO- http://serveur-web
```

On récupère le contenu HTML du serveur. On peut utiliser le nom serveur-web car Docker fournit un DNS interne automatique sur les réseaux personnalisés (User-defined bridge).

4.5

Depuis un conteneur client-externe (réseau par défaut), la connexion échoue. Le réseau bridge par défaut ne supporte pas la résolution DNS par nom de conteneur et est isolé des réseaux personnalisés.

4.6

Pour connecter un conteneur déjà lancé à un réseau :

```
docker network connect mon-reseau client-externe
```

---

Exercice 5 — Containeriser un serveur Flask

5.3

Ordre optimal du Dockerfile :

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["flask", "run", "--host=0.0.0.0"]
```

Raisonnement : On copie et on installe les dépendances avant de copier le reste du code. Comme les dépendances changent rarement par rapport au code source, Docker pourra réutiliser le cache de l'étape RUN pip install lors des prochains builds.

5.5

```
docker run -d -p 5000:5000 -e APP_ENV=production --name flask-app flask-app:v1
```

5.6

Si on ne précise pas APP_ENV, la valeur par défaut définie dans le code Python via os.environ.get("APP_ENV", "développement") s'affiche.

5.7

La taille est réduite grâce à l'image de base slim.
Pistes pour réduire davantage :

Utiliser un multi-stage build (compiler dans une image, ne garder que l'exécutable dans une autre).

Utiliser un fichier .dockerignore pour ne pas envoyer les fichiers inutiles (comme __pycache__ ou les tests) au démon Docker.
