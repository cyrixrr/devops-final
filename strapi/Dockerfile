# =========================
# Strapi multi-stage Dockerfile
# =========================
# Why multi-stage?
# - Stage 1 ("build") installs ALL deps (including dev deps) and builds the Strapi admin UI.
# - Stage 2 ("runtime") installs ONLY production deps and runs the app (smaller + cleaner image).
#
# IMPORTANT (DevOps best practice):
# - Do NOT bake environment selection into the image (e.g., ENV NODE_ENV=production).
#   Instead, pass NODE_ENV (and other config) from Kubernetes/Helm via Deployment env vars.
#   That’s what you already do in your Helm chart values.yaml + templates.

# -------------------------
# Stage 1: Build
# -------------------------
FROM node:20-alpine AS build          # Base image for building (Node 20 on Alpine, small & common)
WORKDIR /app                         # Set working directory inside the container

COPY package*.json ./                # Copy package.json + package-lock.json first (enables Docker layer caching)
RUN npm ci                           # Install dependencies EXACTLY from package-lock.json (reproducible builds)

COPY . .                             # Copy the rest of the Strapi source code into the image
RUN npm run build                    # Build Strapi admin panel/assets (compile frontend, etc.)

# -------------------------
# Stage 2: Runtime
# -------------------------
FROM node:20-alpine                  # Fresh clean runtime image (does NOT inherit build-stage layers)
WORKDIR /app                         # Working directory for the running app

COPY package*.json ./                # Copy package manifests again into runtime stage
RUN npm ci --omit=dev                # Install ONLY production dependencies (smaller image, fewer attack surface)

COPY --from=build /app ./            # Copy the FULL prepared app from build stage (including built admin assets)

EXPOSE 1337                          # Document that Strapi listens on port 1337 (K8s will map it via Service)
CMD ["npm", "run", "start"]          # Start Strapi (NODE_ENV should come from Kubernetes env, not Dockerfile)
