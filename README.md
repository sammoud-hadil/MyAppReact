# TP4 : Docker et Docker Compose — Application MERN

Containerisation d'une application MERN (MongoDB, Express, React, Node.js) avec Docker, puis orchestration des trois conteneurs avec Docker Compose.

## Structure du projet

```
MyAppReact/
├── Client/                 # Application React (Vite)
│   ├── Dockerfile
│   ├── .dockerignore
│   └── src/App.jsx         # appelle http://localhost:9000/api/health
├── Server/                 # API Express
│   ├── Dockerfile
│   ├── .dockerignore
│   └── src/index.mjs       # route GET /api/health + connexion MongoDB
├── docker-compose.yml
└── README.md
```

> Remarque : dans ce dépôt, les dossiers s'appellent `Client` et `Server` (avec une majuscule). Sous Linux et dans Docker les chemins sont sensibles à la casse, donc toutes les commandes utilisent `./Client` et `./Server`.

---

## 1. Cloner le projet

```bash
git clone https://github.com/wiem-kb/MyAppReact.git
cd MyAppReact
```

Le projet contient deux parties : `Client` (front-end React construit avec Vite) et `Server` (API Express qui expose `GET /api/health` sur le port 9000).

## 2. Créer les Dockerfiles

### a. Serveur (Express) — `Server/Dockerfile`

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ENV PORT=9000
ENV MONGO_URI=mongodb://mongodb:27017/mern
EXPOSE 9000
CMD ["npm", "start"]
```

| Instruction | Rôle |
|---|---|
| `FROM node:lts-alpine` | Image de base Node.js LTS, légère (Alpine Linux) |
| `WORKDIR /app` | Répertoire de travail dans le conteneur |
| `COPY package*.json ./` puis `RUN npm install` | Installe les dépendances. Copier ces fichiers en premier permet à Docker de réutiliser le cache tant que les dépendances ne changent pas |
| `COPY . .` | Copie le code source du serveur |
| `ENV MONGO_URI=...` | Valeur par défaut, surchargée par Docker Compose |
| `EXPOSE 9000` | Documente le port utilisé par l'API |
| `CMD ["npm", "start"]` | Lance `node src/index.mjs` |

Construction de l'image :

```bash
docker build -t mern-server ./Server
```

### b. Client (React) — `Client/Dockerfile`

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN rm -f package-lock.json && npm install
COPY . .
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
```

- `npm run build` lance `vite build`, qui génère les fichiers statiques dans `dist/` (Vite utilise `dist/`, pas `build/` comme Create React App).
- `serve` est un serveur HTTP léger. `-s` active le mode SPA et `-l 3000` écoute sur le port 3000.

Construction de l'image :

```bash
docker build -t mern-client ./Client
```

> L'énoncé indique `docker build -t mern-server ./client` pour le client. C'est une coquille : l'image du client doit s'appeler `mern-client`, sinon elle écrase l'image du serveur.

### Fichiers `.dockerignore`

Chaque dossier contient un `.dockerignore` qui exclut `node_modules` (et `dist` côté client). Les dépendances sont réinstallées dans le conteneur pour le bon système (Alpine Linux), et l'image reste plus légère.

## 3. Créer un réseau Docker

```bash
docker network create mern-network
```

Les conteneurs d'un même réseau *bridge* défini par l'utilisateur peuvent communiquer entre eux en utilisant leur nom comme nom d'hôte (DNS interne de Docker).

## 4. Exécuter MongoDB dans un conteneur

```bash
docker run -d --name mongodb --network mern-network mongo
```

MongoDB écoute sur le port 27017 à l'intérieur du réseau. Le port n'est pas publié sur la machine hôte car seul le serveur a besoin d'y accéder.

## 5. Exécuter les conteneurs du serveur et du client

```bash
docker run -d --name server --network mern-network -p 9000:9000 \
  -e MONGO_URI=mongodb://mongodb:27017/mern mern-server

docker run -d --name client --network mern-network -p 3000:3000 mern-client
```

Vérification :

```bash
docker ps
curl http://localhost:9000/api/health   # {"message":"API is running","db":"connected"}
```

Puis ouvrir http://localhost:3000 dans le navigateur : la page affiche « API is running ».

Nettoyage avant de passer à Docker Compose (les noms de conteneurs doivent être libres) :

```bash
docker rm -f client server mongodb
docker network rm mern-network
```

## 6. Docker Compose — `docker-compose.yml`

```yaml
services:
  mongodb:
    image: mongo
    container_name: mongodb
    volumes:
      - mongo-data:/data/db
    networks:
      - mern-network

  server:
    build: ./Server
    image: mern-server
    container_name: server
    ports:
      - "9000:9000"
    environment:
      - MONGO_URI=mongodb://mongodb:27017/mern
    depends_on:
      - mongodb
    networks:
      - mern-network

  client:
    build: ./Client
    image: mern-client
    container_name: client
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
  mongo-data:
```

Points clés :

- **`build`** construit l'image à partir du Dockerfile du dossier indiqué, et **`image`** lui donne un nom (`mern-server`, `mern-client`).
- **`depends_on`** fixe l'ordre de démarrage : `mongodb`, puis `server`, puis `client`.
- **`MONGO_URI=mongodb://mongodb:27017/mern`** : le serveur joint MongoDB en utilisant le **nom du service** `mongodb` comme nom d'hôte.
- **`mern-network`** est un réseau de type `bridge` partagé par les trois services.
- **Volume `mongo-data`** (ajout facultatif) : les données MongoDB sont conservées même si le conteneur est supprimé.

Lancement :

```bash
docker compose up --build
```

Commandes utiles :

```bash
docker compose ps            # état des services
docker compose logs server   # logs du serveur
docker compose down          # arrêter et supprimer les conteneurs
docker compose down -v       # ... et supprimer aussi le volume de données
```

## Modifications apportées au code

1. **Connexion à MongoDB (`Server/src/index.mjs`)** : le serveur d'origine n'utilisait pas MongoDB. J'ai ajouté `mongoose`, qui se connecte avec la variable d'environnement `MONGO_URI`. La route `/api/health` renvoie maintenant aussi l'état de la base (`"db": "connected"` ou `"disconnected"`).
2. **`Client/package-lock.json` régénéré** : le fichier d'origine avait été créé sous Windows et ne contenait que le binaire natif de Vite/rolldown pour Windows (`@rolldown/binding-win32-x64-msvc`). Sous Linux, `npm run build` échouait (`Cannot find native binding`), et donc aussi dans le conteneur Alpine. Le nouveau lock contient les binaires de toutes les plateformes, y compris `linux-x64-musl` pour Alpine.

## Remarque sur la communication client → serveur

Le code React s'exécute **dans le navigateur de l'utilisateur**, pas dans le conteneur. L'appel `fetch("http://localhost:9000/api/health")` fonctionne donc parce que le port 9000 du serveur est publié sur la machine hôte (`-p 9000:9000`). Le nom de service `server` n'est utilisable qu'entre conteneurs (par exemple, server → mongodb).
