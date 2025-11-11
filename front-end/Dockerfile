FROM node:22.18.0-alpine

WORKDIR /app

# Copier les fichiers package d'abord
COPY package*.json ./
COPY vite.config.js ./

# Installer toutes les dépendances (dont devDependencies)
RUN npm install
#  Mise à jour de esbuild
RUN npm install esbuild@latest

# Copier le code source
COPY . .

# Exposer le port que Vite utilise (généralement 5173 ou 3000)
EXPOSE 5173

# Lancer en mode développement
CMD ["npm", "run", "dev", "--", "--host=0.0.0.0"]
