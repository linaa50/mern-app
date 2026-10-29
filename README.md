# TP2 : Docker et Docker Compose

**Module :** DevOps 2026-27
**Étudiante :** Lina NASRI
**Dépôt :** https://github.com/linaa50/mern-app

## 1. Objectifs

- Utiliser Docker pour containeriser des applications.
- Utiliser Docker Compose pour orchestrer plusieurs conteneurs.
- Déployer une application MERN (MongoDB, Express, React, Node.js) avec Docker.
- Assurer la persistance des données, vérifier le fonctionnement des services et adapter la démarche à un projet Micro-services.

## 2. Prérequis

- Docker Desktop (Windows) lancé
- Git
- Un compte GitHub

## 3. Structure du projet

```
mern-app/
├── client/                 # Application React
│   ├── Dockerfile
│   └── .dockerignore
├── server/                 # API Express
│   ├── Dockerfile
│   └── .dockerignore
├── docker-compose.yml
├── .env.example
├── .gitignore
├── screenshots/
└── README.md
```

## 4. Partie 1 : Application MERN

### 4.1 Fork et clonage

Le dépôt d'origine a été forké sur mon compte GitHub, puis mon fork a été cloné (et non le dépôt original) :

```bat
git clone https://github.com/linaa50/mern-app.git
cd mern-app
```

Vérifications faites dans le code :
- le script `start` de `server/package.json` est correct ;
- le serveur se connecte à MongoDB via `process.env.MONGO_URI`.

### 4.2 Dockerfiles

**server/Dockerfile** (image `mern-server`)

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci || npm install
COPY . .
EXPOSE 9000
CMD ["npm", "start"]
```

**client/Dockerfile** (image `mern-client`)

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ARG REACT_APP_API_URL=http://localhost:9000
ENV REACT_APP_API_URL=$REACT_APP_API_URL
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "build", "-l", "3000"]
```

**.dockerignore** (dans `client/` et `server/`)

```
node_modules
npm-debug.log
.env
.git
build
```

Commandes de build :

```bat
cd server
docker build -t mern-server .
cd ..\client
docker build -t mern-client .
cd ..
docker images
```

![docker images](screenshots/01-docker-images.png)

### 4.3 Réseau Docker

```bat
docker network create mern-network
```

### 4.4 MongoDB avec volume nommé

```bat
docker volume create mern-data
docker run -d --name mongodb --network mern-network -v mern-data:/data/db mongo
```

### 4.5 Lancement du serveur et du client

```bat
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mern mern-server
docker run -d --name client --network mern-network -p 3000:3000 mern-client
```

![docker ps](screenshots/02-docker-ps.png)

**Question : pourquoi deux URL différentes pour joindre l'API et MongoDB ?**

| Variable | Valeur | Qui l'utilise | Pourquoi |
|---|---|---|---|
| `REACT_APP_API_URL` | `http://localhost:9000` | Le **navigateur** | Le code React s'exécute dans le navigateur, hors du réseau Docker. Il joint l'API par le port publié sur la machine hôte (`-p 9000:9000`). |
| `MONGO_URI` | `mongodb://mongodb:27017/mern` | Le **serveur Express** (dans un conteneur) | Le serveur est dans le réseau Docker `mern-network`, où le nom du conteneur (`mongodb`) sert de nom d'hôte. |

Utiliser `http://server:9000` dans le navigateur ne marcherait pas, car le navigateur ne connaît pas le DNS interne de Docker. Inversement, `localhost` dans le conteneur du serveur désignerait le conteneur lui-même, pas MongoDB.

De plus, `REACT_APP_API_URL` est intégrée **au moment du build** (`npm run build`), c'est pourquoi elle est passée en `ARG` dans le Dockerfile.

### 4.6 Docker Compose

Les conteneurs lancés à la main ont été supprimés (`docker rm -f client server mongodb`), puis remplacés par `docker-compose.yml` :

```yaml
services:
  mongodb:
    image: mongo
    container_name: mongodb
    restart: unless-stopped
    volumes:
      - mern-data:/data/db
    networks:
      - mern-network
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  server:
    build: ./server
    container_name: server
    restart: unless-stopped
    ports:
      - "9000:9000"
    environment:
      - MONGO_URI=mongodb://mongodb:27017/mern
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - mern-network

  client:
    build:
      context: ./client
      args:
        REACT_APP_API_URL: http://localhost:9000
    container_name: client
    restart: unless-stopped
    ports:
      - "3000:3000"
    depends_on:
      - server
    networks:
      - mern-network

networks:
  mern-network:
    driver: bridge

volumes:
  mern-data:
```

Points importants :
- **Réseau** `mern-network` de type `bridge`.
- **Volume nommé** `mern-data` pour conserver les données de MongoDB.
- **Politique de redémarrage** : `restart: unless-stopped`.
- **Healthcheck** sur MongoDB (ping toutes les 10 s).
- **Ordre de démarrage** : `server` attend que MongoDB soit `healthy` (`condition: service_healthy`), puis `client` attend `server`.

### 4.7 Commandes principales

| Action | Commande |
|---|---|
| Construction et démarrage | `docker compose up --build` |
| Démarrage en arrière-plan | `docker compose up -d` |
| État des services | `docker compose ps` |
| Journaux du serveur | `docker compose logs --tail=50 server` |
| Arrêt (données conservées) | `docker compose down` |
| Arrêt + suppression des volumes | `docker compose down -v` (efface les données) |

### 4.8 Tests effectués

1. `docker compose up --build` : les trois services démarrent, MongoDB indique « Waiting for connections » sur le port 27017.
2. `docker compose ps` : `mongodb` est `healthy`, `server` et `client` sont `Up`.
3. Ouverture de `http://localhost:3000` : la page **Record List** s'affiche.

![Page initiale](screenshots/03-app-vide.png)

4. Test complet React, Express, MongoDB : création d'un enregistrement via **Create Record**. Il apparaît dans la liste.

![Enregistrement créé](screenshots/04-app-avec-donnee.png)

5. Test de persistance : `docker compose down` puis `docker compose up -d`. Après rechargement de la page, l'enregistrement est toujours présent, car le volume `mern-data` n'a pas été supprimé.

![Après redémarrage](screenshots/05-apres-redemarrage.png)

![docker compose ps](screenshots/06-compose-ps.png)

![Logs du serveur](screenshots/07-logs-server.png)

### 4.9 Question : comment diagnostiquer un service qui échoue au démarrage ?

1. **Voir l'état** : `docker compose ps -a` indique si le conteneur est `Exited`, en `Restarting`, ou `unhealthy`, avec son code de sortie.
2. **Lire les journaux** : `docker compose logs <service>` (ou `--tail=50`, ou `-f` pour suivre en direct). L'erreur y est presque toujours visible.
3. **Inspecter le conteneur** : `docker inspect <conteneur>` pour voir la configuration, l'état de santé, les variables d'environnement et le réseau.
4. **Entrer dans le conteneur** : `docker exec -it <conteneur> sh` pour tester à l'intérieur (variables, connectivité, fichiers). Si le conteneur est arrêté, utiliser `docker compose run --rm <service> sh`.
5. **Vérifier les causes fréquentes** :
   - `MONGO_URI` absente ou erronée ;
   - script `npm start` incorrect dans `package.json` ;
   - port déjà utilisé sur la machine ;
   - MongoDB pas encore prêt (vérifier le healthcheck et `depends_on`) ;
   - service sur un mauvais réseau ;
   - `REACT_APP_API_URL` mal définie au build.
6. **Valider le fichier Compose** : `docker compose config`.
7. **Reconstruire sans cache** si besoin : `docker compose build --no-cache`.

## 5. Partie 2 : Projet Micro-services

> À compléter avec les informations de ton projet (noms des services, langages, ports).

### 5.1 Architecture

| Service | Langage / techno | Port | Base de données |
|---|---|---|---|
| service-a | *(à compléter)* | *(à compléter)* | *(à compléter)* |
| service-b | *(à compléter)* | *(à compléter)* | *(à compléter)* |
| gateway (si présent) | *(à compléter)* | *(à compléter)* | - |

### 5.2 Fichiers de conteneurisation

- Un `Dockerfile` et un `.dockerignore` dans chaque microservice.
- Un `docker-compose.yml` à la racine qui démarre tous les services, leurs bases de données et leurs dépendances.
- Configuration : ports, réseau commun, variables d'environnement, `depends_on`, volumes persistants.

### 5.3 Commandes

| Action | Commande |
|---|---|
| Construction et démarrage | `docker compose up --build` |
| État | `docker compose ps` |
| Diagnostic | `docker compose logs <service>` |
| Arrêt | `docker compose down` |

### 5.4 Scénario de bout en bout testé

*(à compléter, exemple)*
1. Création d'une ressource via l'API (ou la gateway).
2. Le service concerné appelle un autre service par son nom (`http://service-a:PORT`).
3. Vérification en base de données que la donnée est bien enregistrée.

![Scénario micro-services](screenshots/08-microservices.png)

## 6. Difficultés rencontrées et solutions

| Difficulté | Solution |
|---|---|
| Confusion entre `localhost` et le nom du service | `localhost:9000` pour le navigateur (`REACT_APP_API_URL`), `mongodb` pour la communication entre conteneurs (`MONGO_URI`). |
| `REACT_APP_API_URL` non prise en compte | Cette variable est lue au build : elle est passée avec `ARG` / `ENV` avant `npm run build`. |
| Le serveur démarrait avant MongoDB | Ajout d'un `healthcheck` sur MongoDB et de `depends_on` avec `condition: service_healthy`. |
| Build du client très long (plus de 4 minutes) | Normal pour `npm install` d'une application React ; les étapes déjà faites sont mises en cache aux builds suivants. |
| Perte de données à l'arrêt | Utilisation d'un volume nommé `mern-data` et `docker compose down` sans `-v`. |
| Commandes multi-lignes sous Windows | Les commandes `docker run` ont été écrites sur une seule ligne dans l'invite de commandes. |

## 7. Sécurité

Aucun mot de passe ni secret n'est publié dans le dépôt. Les variables nécessaires sont documentées dans `.env.example`, et `.env` est ignoré par Git.

```
MONGO_URI=mongodb://mongodb:27017/mern
REACT_APP_API_URL=http://localhost:9000
```