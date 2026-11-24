# Image de base légère Node.js (version LTS sur Alpine)
FROM node:lts-alpine

# Répertoire de travail dans le conteneur
WORKDIR /app

# Copier d'abord package.json et package-lock.json (meilleur cache Docker)
COPY package*.json ./

# Installer les dépendances
RUN npm install

# Copier le reste des fichiers du serveur
COPY . .

# Variables d'environnement (MONGO_URI est surchargée par Docker Compose)
ENV PORT=9000
ENV MONGO_URI=mongodb://mongodb:27017/mern

# Port exposé par l'API Express
EXPOSE 9000

# Démarrer l'application
CMD ["npm", "start"]
