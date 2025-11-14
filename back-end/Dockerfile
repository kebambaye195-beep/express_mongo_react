FROM node:22.18.0-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 5001

ENV NODE_ENV=development

CMD ["npm", "start"]
