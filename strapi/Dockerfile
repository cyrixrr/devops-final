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

# Base image for building (Node 20 on Alpine, small & common)
FROM node:20-alpine AS build

# Set working directory inside the container
WORKDIR /app

# Copy package.json + package-lock.json first (enables Docker layer caching)
COPY package*.json ./

# Install dependencies EXACTLY from package-lock.json (reproducible builds)
RUN npm ci

# Copy the rest of the Strapi source code into the image
COPY . .

# Build Strapi admin panel/assets (compile frontend, etc.)
RUN npm run build

# -------------------------
# Stage 2: Runtime
# -------------------------

# Fresh clean runtime image (does NOT inherit build-stage layers)
FROM node:20-alpine

# Working directory for the running app
WORKDIR /app

# Copy package manifests again into runtime stage
COPY package*.json ./

# Install ONLY production dependencies (smaller image, fewer attack surface)
RUN npm ci --omit=dev

# Copy the FULL prepared app from build stage (including built admin assets)
COPY --from=build /app ./

# Document that Strapi listens on port 1337 (K8s will map it via Service)
EXPOSE 1337

# Start Strapi (NODE_ENV should come from Kubernetes env, not Dockerfile)
CMD ["npm", "run", "start"]
