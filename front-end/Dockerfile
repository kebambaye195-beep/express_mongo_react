# Frontend Dockerfile
FROM node:22.18.0-alpine

WORKDIR /app

# Copier package.json + package-lock.json + vite.config.js
COPY package*.json package-lock.json vite.config.js ./

# Installer toutes les dépendances (prod + dev) pour Vite
RUN npm ci

# Copier le reste du code source
COPY . .

# Mise à jour optionnelle de esbuild
RUN npm install esbuild@latest

# Exposer le port de Vite
EXPOSE 5173

# Lancer Vite en mode dev
CMD ["npm", "run", "dev", "--", "--host=0.0.0.0"]
