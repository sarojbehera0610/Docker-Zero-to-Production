# ============================================================
# Dockerfile: Java 17 + Spring Boot — Enterprise Java App
# Base Image: eclipse-temurin:17-jdk-alpine (OpenJDK)
# Use case: Spring Boot microservices, enterprise backends,
#           Kafka consumers, gRPC services
# ============================================================

# Stage 1: Build with Maven
FROM maven:3.9-eclipse-temurin-17-alpine AS build

LABEL maintainer="saroj@sarojops.cloud"

WORKDIR /build

# Copy Maven POM first (caches dependency download layer)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src/ ./src/
RUN mvn package -DskipTests -B
# Output: target/app.jar

# Stage 2: Minimal runtime image
FROM eclipse-temurin:17-jre-alpine AS runtime

LABEL description="Spring Boot production service"

# Install curl for health checks
RUN apk add --no-cache curl

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring

WORKDIR /app

# Copy JAR from build stage
COPY --from=build /build/target/*.jar app.jar
RUN chown spring:spring app.jar

USER spring

# JVM tuning: limits memory usage in containers
ENV JAVA_OPTS="-Xms256m -Xmx512m -XX:+UseContainerSupport"

# Spring Boot profile (switch: dev, staging, prod)
ENV SPRING_PROFILES_ACTIVE=production

EXPOSE 8080

# Health check using Spring Actuator endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# ENTRYPOINT: always runs (can't be overridden by docker run args)
# CMD: default args passed to ENTRYPOINT (CAN be overridden)
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]