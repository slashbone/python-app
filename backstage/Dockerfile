# ---------- Stage 1: Builder ----------
FROM node:20-bullseye AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 make g++ pkg-config libssl-dev git ca-certificates \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app
RUN corepack enable

# 1) Copie manifests + a pasta .yarn (para trazer releases/plugins do Yarn)
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn/ .yarn/

# (Opcional, melhora cache de dependências para workspaces)
COPY packages/backend/package.json ./packages/backend/package.json
COPY packages/app/package.json ./packages/app/package.json

# 2) Agora o yarn install encontra o yarn-4.4.1.cjs do projeto
RUN yarn install --immutable

# 3) Só depois copie o restante do código
COPY . .

# 4) Build do projeto
RUN yarn tsc
RUN yarn build:backend

# ---------- Stage 2: Runtime ----------
FROM node:20-bullseye
WORKDIR /app
RUN corepack enable
COPY --from=builder /app/packages/backend/dist ./packages/backend/dist
COPY --from=builder /app/app-config.yaml ./
COPY --from=builder /app/app-config.local.yaml ./
COPY --from=builder /app/app-config.production.yaml ./
COPY packages/backend/package.json ./packages/backend/package.json
COPY yarn.lock ./
RUN yarn install --production
EXPOSE 7007
CMD ["node", "packages/backend/dist/index.cjs"]
