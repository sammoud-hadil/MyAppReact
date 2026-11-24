# Image de base légère Node.js (version LTS sur Alpine)
FROM node:lts-alpine

# Répertoire de travail dans le conteneur
WORKDIR /app

# Copier les fichiers package*.json
COPY package*.json ./

# Installer les dépendances
# (le package-lock.json du dépôt a été généré sous Windows : on le supprime
#  pour que npm installe les bons binaires pour Linux/Alpine)
RUN rm -f package-lock.json && npm install

# Copier le reste des fichiers du client
COPY . .

# Construire l'application React (Vite génère le dossier dist/)
RUN npm run build

# Installer un serveur HTTP léger pour servir les fichiers statiques
RUN npm install -g serve

# Port exposé par le client
EXPOSE 3000

# Servir le dossier de build sur le port 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
