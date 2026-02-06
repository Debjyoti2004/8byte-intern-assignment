# STAGE 1
FROM node:18-alpine AS builder

WORKDIR /app
    
COPY package*.json ./
    
RUN npm ci
    
COPY . .
    
# STAGE 2
FROM node:18-alpine AS runner
    
ENV NODE_ENV=production
    
WORKDIR /app

COPY --from=builder --chown=node:node /app/package.json ./
COPY --from=builder --chown=node:node /app/node_modules ./node_modules
COPY --from=builder --chown=node:node /app/app.js ./
    
EXPOSE 3000
    
USER node

CMD ["node", "app.js"]