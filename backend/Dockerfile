# ===== ÉTAPE 1 : BUILD =====
# On utilise Maven pour compiler le code Java
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

# Copier d'abord pom.xml seul (optimisation cache Docker)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copier le reste du code et compiler
COPY src ./src
RUN mvn clean package -DskipTests -B

# ===== ÉTAPE 2 : RUN =====
# Image finale légère — juste Java, pas Maven
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Créer un utilisateur non-root pour la sécurité
RUN addgroup -S uaszgpt && adduser -S uaszgpt -G uaszgpt

# Copier le JAR compilé depuis l'étape build
COPY --from=build /app/target/*.jar app.jar

# Donner les droits à l'utilisateur
RUN chown uaszgpt:uaszgpt app.jar
USER uaszgpt

# Port exposé par Spring Boot
EXPOSE 8080

# Health check — vérifie que l'app répond
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -q -O- http://localhost:8080/actuator/health || exit 1

# Démarrer l'application
ENTRYPOINT ["java", "-jar", "app.jar"]