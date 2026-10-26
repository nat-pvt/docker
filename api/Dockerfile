FROM node:22-alpine

WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./

RUN npm ci

COPY src ./src

EXPOSE 3000

USER node

CMD ["node", "src/server.js"]