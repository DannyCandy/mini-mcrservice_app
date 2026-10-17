FROM node:22-alpine

ENV NODE_ENV=production
WORKDIR /app

# Cài dependencies trước để tận dụng cache layer khi chỉ sửa code
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force

COPY --chown=node:node . .

USER node

EXPOSE 3001
CMD ["node", "index.js"]
