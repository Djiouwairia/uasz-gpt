# ===== ÉTAPE 1 : BUILD =====
FROM node:20-alpine AS build

WORKDIR /app

# Copier package.json d'abord (optimisation cache)
COPY package*.json .
RUN npm install --silent

# Copier le code et builder
COPY . .
RUN npm run build

# ===== ÉTAPE 2 : RUN =====
# Nginx sert les fichiers statiques — ultra léger
FROM nginx:alpine

# Supprimer la config Nginx par défaut
RUN rm -rf /usr/share/nginx/html/*

# Copier le build React
COPY --from=build /app/build /usr/share/nginx/html

# Config Nginx pour React Router (gère les routes /quiz, /chat, etc.)
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -q -O- http://localhost:80 || exit 1

CMD ["nginx", "-g", "daemon off;"]