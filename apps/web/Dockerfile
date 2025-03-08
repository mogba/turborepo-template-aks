FROM node:22-alpine AS base

# STAGE 0: Install framework dependencies and set up environment

FROM base AS deps

# Enable corepack to make pnpm available
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable

# Set working directory
WORKDIR /app

# Replace <your-major-version> with the major version installed in your repository. For example:
# RUN pnpm add -g turbo@<your-major-version>
RUN pnpm add -g turbo@^2.4.4

# STAGE 1: Generate a partial monorepo with a pruned lockfile for a target workspace

FROM deps AS builder

RUN corepack enable

RUN apk update
RUN apk add --no-cache libc6-compat

WORKDIR /app

COPY . .

# Generate a partial monorepo with a pruned lockfile for a target workspace
# Assuming "web" is the name entered in the project's package.json: { name: "web" }
RUN turbo prune web --docker

# STAGE 2: Install project dependencies and build the project

# Add lockfile and package.json's of isolated subworkspace
FROM deps AS installer

RUN corepack enable

RUN apk update
RUN apk add --no-cache libc6-compat

WORKDIR /app

# First install the dependencies (as they change less often)
COPY --from=builder /app/out/json/ .
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store pnpm fetch --frozen-lockfile
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store pnpm install --frozen-lockfile

# Build the project
# TODO: The build command should be optimized with remote caching once it is setup
COPY --from=builder /app/out/full/ .
# `pnpm run build` command runs `turbo run build` from package.json's build script
RUN pnpm run build

# STAGE 3: Copy the build output and run the app

FROM base AS runner

WORKDIR /app

# Don't run production as root
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
USER nextjs

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
# COPY --from=installer --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=installer --chown=nextjs:nodejs /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=installer --chown=nextjs:nodejs /app/apps/web/public ./apps/web/public

CMD node apps/web/server.js