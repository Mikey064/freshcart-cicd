# STAGE 1: Build Stage
FROM node:22-alpine AS builder
WORKDIR /app

# To optimize layer caching: Copy dependencies first
COPY package*.json ./
RUN npm ci

# Copy app source code and compile
COPY . .

RUN npm run build

# STAGE 2: Runtime Stage 
FROM node:22-alpine AS final

WORKDIR /app

ENV  NODE_ENV=production

# Create a non-root user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy pakage files
COPY package*.json ./

# Install only production dependencies
RUN npm install -g npm@latest && \
    npm ci --omit=dev

# Copy compiled application
COPY --from=builder /app/dist ./dist

USER appuser

EXPOSE 3000

CMD ["node", "dist/index.js"]