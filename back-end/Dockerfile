# Image node alpine
FROM node:22.18.0-alpine

# Dossier de travail
WORKDIR /app

# Copie uniquement les fichiers nécessaires pour installer les dépendances
COPY package*.json ./
COPY .env ./

# Installation (production uniquement)
RUN npm install --production

# Copie du reste du code
COPY . .

# Exposition du port Backend
EXPOSE 5001

# Environnement production
ENV NODE_ENV=production

# Commande de démarrage
CMD ["npm", "start"]
