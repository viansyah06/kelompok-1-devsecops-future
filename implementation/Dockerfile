# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app
COPY app/package.json ./
RUN npm install --production
COPY app/ ./

# Stage 2: Production image
FROM node:18-alpine

# Metadata labels untuk traceability
LABEL org.opencontainers.image.title="supply-chain-demo"
LABEL org.opencontainers.image.description="Demo app for Supply Chain Security Final Project"
LABEL org.opencontainers.image.source="https://github.com/viansyah06/kelompok-1-devsecops-future"

WORKDIR /app
COPY --from=builder /app ./

# Jalankan sebagai non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "index.js"]
