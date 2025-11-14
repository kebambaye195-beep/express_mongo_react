FROM node:22.18.0-alpine

WORKDIR /app

# Copier package.json et installer les dépendances
COPY package*.json ./
RUN npm install

# Copier le code
COPY . .

# Exposer Vite
EXPOSE 5173

# Lancer Vite en mode DEV
CMD ["npm", "run", "dev", "--", "--host=0.0.0.0"]
