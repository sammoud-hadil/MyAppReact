# TP4 – Docker et Docker Compose

Dans ce TP, j'ai containerisé une application MERN (MongoDB, Express, React, Node.js). J'ai d'abord lancé chaque conteneur à la main avec Docker, puis j'ai tout regroupé avec Docker Compose.

## Le projet

J'ai forké le dépôt https://github.com/wiem-kb/MyAppReact puis je l'ai cloné :

```bash
git clone https://github.com/sammoud-hadil/MyAppReact.git
cd MyAppReact
```

Il contient deux dossiers :
- `Client` : l'application React (faite avec Vite)
- `Server` : l'API Express, avec une route `/api/health` sur le port 9000

Attention, les dossiers commencent par une majuscule (`Client` et `Server`), donc dans les commandes j'ai utilisé `./Client` et `./Server` et pas `./client` comme dans l'énoncé.

## Dockerfile du serveur

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

On part de l'image `node:lts-alpine` qui est légère. Je copie d'abord les `package*.json` et je fais `npm install` avant de copier le reste du code : comme ça Docker garde les dépendances en cache si je modifie seulement le code. La variable `MONGO_URI` a une valeur par défaut mais c'est Docker Compose qui la définit vraiment.

```bash
docker build -t mern-server ./Server
```

## Dockerfile du client

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

`npm run build` génère les fichiers dans le dossier `dist` (avec Vite c'est `dist` et pas `build`). Ensuite j'utilise `serve` pour servir ces fichiers sur le port 3000.

```bash
docker build -t mern-client ./Client
```

Remarque : dans l'énoncé, la commande pour le client est `docker build -t mern-server ./client`. J'ai mis `mern-client` sinon l'image du client remplace celle du serveur.

J'ai aussi ajouté un fichier `.dockerignore` dans chaque dossier pour ne pas copier `node_modules` dans l'image.

## Lancer les conteneurs à la main

Création du réseau :

```bash
docker network create mern-network
```

MongoDB :

```bash
docker run -d --name mongodb --network mern-network mongo
```

Serveur et client :

```bash
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mern mern-server
docker run -d --name client --network mern-network -p 3000:3000 mern-client
```

Comme les trois conteneurs sont sur le même réseau, le serveur peut joindre MongoDB juste avec le nom `mongodb`.

Avant de passer à Docker Compose, j'ai supprimé ces conteneurs :

```bash
docker rm -f client server mongodb
docker network rm mern-network
```

## Docker Compose

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

- `depends_on` permet de démarrer MongoDB d'abord, puis le serveur, puis le client.
- Le serveur se connecte à MongoDB avec `mongodb://mongodb:27017/mern` : `mongodb` c'est le nom du service.
- J'ai ajouté un volume `mongo-data` pour ne pas perdre les données si le conteneur est supprimé.

Pour tout lancer :

```bash
docker compose up --build
```

Et pour arrêter :

```bash
docker compose down
```

## Résultat

<img width="1920" height="1200" alt="Application lancée avec Docker Compose" src="https://github.com/user-attachments/assets/8f120c3e-cf81-4853-b9b4-f4b427b189c8" />

- http://localhost:3000 affiche « API is running »
- http://localhost:9000/api/health renvoie `{"message":"API is running","db":"connected"}`

## Problèmes rencontrés

**Connexion à MongoDB** : au départ le serveur n'était pas connecté à MongoDB. J'ai ajouté `mongoose` dans `Server/src/index.mjs` pour se connecter avec la variable `MONGO_URI`. J'ai aussi modifié la route `/api/health` pour qu'elle affiche si la base est connectée ou pas.

**Erreur au build du client** : le `package-lock.json` du client avait été généré sous Windows, donc `npm run build` plantait dans le conteneur Linux avec l'erreur `Cannot find native binding`. Pour régler ça, le Dockerfile supprime ce fichier avant `npm install`, et npm installe alors les bonnes versions pour Linux.
