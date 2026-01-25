# Build stage
FROM node:20-alpine AS build
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
ENV NODE_ENV=production
RUN npm run build

# Runtime stage
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev

COPY --from=build /app ./

EXPOSE 1337
CMD ["npm", "run", "start"]
