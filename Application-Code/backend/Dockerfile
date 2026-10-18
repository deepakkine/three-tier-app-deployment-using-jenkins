FROM node:20-alpine

ENV NODE_ENV=production
WORKDIR /usr/src/app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY db.js index.js ./
COPY models ./models
COPY routes ./routes

USER node

EXPOSE 3500

CMD ["node", "index.js"]
