# syntax=docker/dockerfile:1

# ---- build: install all deps, build the React app ----
FROM oven/bun:1 AS build
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

# ---- run: production deps + server source + built UI ----
FROM oven/bun:1 AS run
WORKDIR /app
ENV NODE_ENV=production \
    PORT=3000 \
    DATA_DIR=/data

COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production

COPY --from=build /app/dist ./dist
COPY server ./server
COPY shared ./shared

# SQLite lives here; mount a volume at /data
RUN mkdir -p /data && chown -R bun:bun /data /app
VOLUME /data
USER bun

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD bun -e "fetch('http://localhost:' + (process.env.PORT ?? 3000) + '/api/health').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"

CMD ["bun", "server/index.ts"]
