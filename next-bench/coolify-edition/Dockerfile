# syntax=docker.io/docker/dockerfile:1

###########################################################################
# 🔹 Base image
###########################################################################
FROM node:20-alpine AS base

# Fix common runtime deps
RUN apk add --no-cache libc6-compat tini

WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

###########################################################################
# 🔹 Dependencies
###########################################################################
FROM base AS deps

# Copy package manifests
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* .npmrc* ./

# Install deps depending on lockfile and PM
RUN \
  if [ -f yarn.lock ]; then \
      yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then \
      npm ci; \
  elif [ -f pnpm-lock.yaml ]; then \
      corepack enable pnpm && pnpm i --frozen-lockfile; \
  else \
      echo "❌ No lockfile found." && exit 1; \
  fi

###########################################################################
# 🔹 Build
###########################################################################
FROM base AS builder

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# If you use "output: standalone" in next.config.js this makes the smallest bundle
RUN \
  if [ -f yarn.lock ]; then \
      yarn run build; \
  elif [ -f package-lock.json ]; then \
      npm run build; \
  elif [ -f pnpm-lock.yaml ]; then \
      corepack enable pnpm && pnpm run build; \
  else \
      echo "❌ No lockfile found." && exit 1; \
  fi

###########################################################################
# 🔹 Runner
###########################################################################
FROM base AS runner

# Security: dedicated user
RUN addgroup --system --gid 1001 nodejs \
  && adduser --system --uid 1001 nextjs

USER nextjs

# Bring in all necessary runtime parts
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./package.json

# 👇 Install PM2 globally, very light on alpine
RUN npm install -g pm2@5

EXPOSE 3000

# Optional: auto‑restart on memory pressure (>500MB), single‑instance fallback
# PM2 spawns multiple workers on multi‑core CPUs (cluster mode)
ENTRYPOINT ["/sbin/tini", "--"]

CMD ["pm2-runtime", "--name", "next-app", "--max-memory-restart", "500M", "--no-autorestart", "--", "node", "server.js"]