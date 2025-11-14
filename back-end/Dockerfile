# Backend Dockerfile
FROM node:22.18.0-alpine

WORKDIR /app

# Copier uniquement package.json + package-lock.json pour un scan Trivy correct
COPY package*.json package-lock.json ./

# Installer uniquement les dépendances de production pour réduire la taille
RUN npm ci --only=production

# Copier le reste du code
COPY . .

# Définir NODE_ENV à production
ENV NODE_ENV=production

# Exposer le port
EXPOSE 5001

# Commande pour démarrer l'application
CMD ["npm", "start"]
