# TP2 - Optimiser une image Docker Debian 12

## 🎯 Objectifs

- Réduire la **taille d'une image Docker** existante sans casser l’application.
- Appliquer pas à pas les **bonnes pratiques d’optimisation**.
- **Contrôler l'impact** de chaque modification avec `docker images` et `dive`.

👉 Ce TP fait suite à [`01-debian12-basique`](../01-debian12-basique). On repart
de cette base pour l’optimiser.

## 📁 Structure du projet

```bash
02-debian12-optimisee/
├── app/
│   ├── main.py
│   └── requirements.txt
├── .dockerignore
├── dockerfile
```

## Etape 0 - On fait le point

🔍 **Avant de commencer**, vérifiez la taille de l’image de base :

Si besoin reconstruisez l’image de base :

```bash
docker build -t fastapi-auto .
```

On controle la taille de l’image :

```bash
docker images | grep fastapi-auto
REPOSITORY     TAG       IMAGE ID       CREATED          SIZE
fastapi-auto   latest    7e2e921dad53   14 seconds ago   678MB
```

678 Mo, c’est déjà pas mal pour une image de base.
On va essayer de réduire ça.

## 🧱 Étape 1 - On regroupe les commandes RUN

Pour limiter le nombre de couches, on va **regrouper les commandes `RUN`** dans le
Dockerfile. Et on demande de ne pas installer les paquets « recommandés » par
Debian.

**Avant** :

```dockerfile
RUN apt-get update
RUN apt-get install -y python3 python3-pip
```

**Après** :

```dockerfile
RUN apt-get update && \
    apt-get install -y python3 python3-pip
```

## Étape 2 - Activer `--no-install-recommends`

👉 On empêche Debian d’installer des paquets « recommandés » inutiles.

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip
```

✅ **Rebuild** :

```bash
docker build -t fastapi-optimise:v1 .
```

Vous remarquez que l’image se reconstruit depuis le début, car on a modifié le
Dockerfile.

✅ **Contrôle de la taille de l'image** :

```bash
docker images

fastapi-optimise   v1        d1c77c5394e7   7 seconds ago        256MB
fastapi-auto       latest    7e2e921dad53   6 minutes ago        678MB
```

On teste que l’API fonctionne toujours :

```bash
docker run -d -p 8000:8000 fastapi-optimise:v1
```

Un petit `curl` pour vérifier que l’API fonctionne :

```bash
curl http://localhost:8000/health
{"status":"ok"}
```

Aucune perte de fonctionnalité, mais une image déja plus petite.

On stoppe le conteneur :

```bash
docker ps
docker stop $(docker ps -q)
```

Cette commande arrête tous les conteneurs en cours d’exécution.

## 🧹 Étape 2 - Nettoyer le cache APT

On va nettoyer le cache APT après l’installation des paquets.

Ajoutez la commande suivante dans le RUN apt :

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
```

Cela évite que Docker garde dans l’image :

- Les index des paquets APT
- Les fichiers temporaires d’installation

On reconstruit l’image :

```bash
docker build -t fastapi-optimise:v2 .
```

✅ **Contrôle de la taille de l'image** :

```bash
docker images | grep fastapi
fastapi-optimise   v2        7e2e921dad53   6 minutes ago        256MB
fastapi-auto       latest    7e2e921dad53   6 minutes ago        678MB
```

On remarque que la taille de l’image n’a pas changé. C’est normal, car Docker
ne reconstruit pas les couches qui n’ont pas changé. On va forcer la
reconstruction de l’image.

```bash
docker build --no-cache -t fastapi-optimise:v2 .
```

✅ **Contrôle de la taille de l'image** :

```bash
docker images | grep fastapi

REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
fastapi-optimise   v1        d1c77c5394e7   4 minutes ago    256MB
fastapi-optimise   v2        ac5a57858d17   5 minutes ago    236MB
fastapi-auto       latest    7e2e921dad53   10 minutes ago   678MB
```

On remarque que la taille de l’image a bien diminué.

On teste que l’API fonctionne toujours :

```bash
docker run -d -p 8000:8000 fastapi-optimise:v2
```

On relance le test :

```bash
curl http://localhost:8000/health
{"status":"ok"}
```

On stoppe le conteneur :

```bash
docker stop $(docker ps -q)
```

## 📦 Étape 3 - Créer un `.dockerignore`

Ajoutez un fichier `.dockerignore` à la racine :

```txt
__pycache__/
*.pyc
*.pyo
*.md
.env
.dive-ci
.dive.yml
```

✅ Cela évite que des fichiers inutiles (tests, logs, markdown, caches Python…) soient copiés dans l’image.

🔄 **Rebuild** :

```bash
docker build -t fastapi-optimise:v3 .
```

✅ **Vérifiez que les fichiers ignorés ne sont pas dans l’image** :

```bash
docker run -it fastapi-optimise:v2 sh
ls /app
```

## 🔍 Étape 4 - Regroupement de toutes les commandes RUN

Il reste une commande `RUN` dans le Dockerfile :

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

On va la regrouper avec les autres commandes `RUN` pour réduire le nombre de
couches. Ajoutez la commande `pip` dans le même `RUN` :

```dockerfile
# Copie de l’application dans le conteneur
WORKDIR /app
COPY app/ /app/

# Installation de Python 3 et pip
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    pip3 install --break-system-packages -r requirements.txt
```

J'ai aussi décidé de déplacer la commande `COPY` avant le `RUN` pour que Docker
puisse mettre en cache la couche de l’application. Cela évite de reconstruire
l’image à chaque fois qu’on modifie le code de l’application.

Cette fois on force la reconstruction de l’image :

```bash
docker build --no-cache -t fastapi-optimise:v4 .
```

On controle la taille de l’image :

```bash
docker images | grep fastapi
REPOSITORY         TAG       IMAGE ID       CREATED          SIZE
fastapi-optimise   v4        bb2121d01538   5 seconds ago        236MB
fastapi-optimise   v1        d1c77c5394e7   16 minutes ago       256MB
fastapi-optimise   v2        ac5a57858d17   17 minutes ago       236MB
fastapi-optimise   v3        ac5a57858d17   17 minutes ago       236MB
fastapi-auto       latest    7e2e921dad53   22 minutes ago       678MB
```

On remarque que la taille de l’image n’a pas changé. On a atteint un bon
compromis entre la taille de l’image et le nombre de couches.
On teste que l’API fonctionne toujours :

```bash
docker run -d -p 8000:8000 fastapi-optimise:v4
```

On relance le test :

```bash
curl http://localhost:8000/health
{"status":"ok"}
```

On stoppe le conteneur :

```bash
docker stop $(docker ps -q)
```

## 🧠 Ce que vous avez appris

Vous avez appris à :

- Réduire la taille d’une image Docker en optimisant le `dockerfile`.
- Utiliser des techniques comme `--no-install-recommends`, le nettoyage APT et
  `.dockerignore`.
- Analyser la taille des images avec `docker images` et `dive`.
- Regrouper les commandes `RUN` pour réduire le nombre de couches.
- Utiliser `docker build --no-cache` pour forcer la reconstruction de l’image.
- Vérifier que l’application fonctionne toujours après les modifications.
- Utiliser `docker run -it` pour explorer l’image et vérifier les fichiers
  présents.
- Utiliser `docker stop` pour arrêter les conteneurs en cours d’exécution.
