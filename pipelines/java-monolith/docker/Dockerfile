# ============================================================
# Stage 1 — Build
# Java 21 (current LTS) with Maven 3.9.9 on Alpine
# ============================================================
FROM maven:3.9.9-eclipse-temurin-21-alpine AS builder

WORKDIR /usr/src/app

# Copy pom.xml first to leverage Docker layer caching for dependencies
COPY pom.xml .

# Resolve dependencies separately from source changes for better caching
RUN mvn dependency:resolve

# Copy source and build
COPY src ./src
RUN mvn clean package -DskipTests

# ============================================================
# Stage 2 — Production Runtime
# JRE-only image: smaller size + reduced attack surface vs JDK
# ============================================================
FROM eclipse-temurin:21-jre-alpine

# OCI standard image labels
LABEL maintainer="github.com/ibtisam-iq"
LABEL org.opencontainers.image.title="BankApp"
LABEL org.opencontainers.image.description="Banking Web Application — DevSecOps Portfolio Project"
LABEL org.opencontainers.image.source="https://github.com/ibtisam-iq/java-app-multi-deployment"
LABEL org.opencontainers.image.licenses="MIT"

WORKDIR /usr/src/app

# Security hardening: run as non-root user
# Running as root inside a container is a critical vulnerability flagged by Trivy/Snyk
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy built JAR from builder stage
COPY --from=builder /usr/src/app/target/*.jar app.jar

# Set correct ownership before switching user
RUN chown appuser:appgroup app.jar

USER appuser

EXPOSE 8000

# Health check using Spring Boot Actuator endpoint
# Requires spring-boot-starter-actuator in pom.xml
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:8000/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
