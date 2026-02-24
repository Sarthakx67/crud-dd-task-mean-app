# Backend Dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

COPY package.json package-lock.json* ./

RUN npm install 

COPY . .

ENV NODE_ENV=production
EXPOSE 8080

CMD ["node", "server.js"]
