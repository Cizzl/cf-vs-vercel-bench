# Stage 1 — Build
FROM node:20-slim AS builder

# Avoid interactive prompts and speed up builds
ENV DEBIAN_FRONTEND=noninteractive \
  NODE_ENV=production \
  NEXT_TELEMETRY_DISABLED=1

WORKDIR /app

# Install core OS deps (no-cache)
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates git && \
    rm -rf /var/lib/apt/lists/*

# Copy only what's needed to install dependencies (leverages caching)
COPY package.json package-lock.json* pnpm-lock.yaml* yarn.lock* ./
# You can use your preferred package manager
RUN npm ci --omit=dev

# Copy rest of source
COPY . .

# Make sure Next is built for production
RUN npm run build

# Stage 2 — Runtime
FROM node:20-slim AS runner

# Lightweight runtime env
ENV NODE_ENV=production \
  NEXT_TELEMETRY_DISABLED=1

WORKDIR /app

# Copy only production files
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/node_modules ./node_modules

# Install PM2 globally (process manager w/ clustering)
RUN npm install -g pm2@5

# Expose Next.js default port
EXPOSE 3000

# Start with PM2 in cluster mode using all CPU cores
# You can limit with PM2_MAX_MEMORY_RESTART to auto-restart on memory pressure
ENTRYPOINT [ "pm2-runtime", "npm", "--", "start", "--", "-p", "3000" ]