# My Steps in Deploying Backend (Spring Boot) with PostgreSQL (Supabase) to Render

## Table of Contents
- [Application Preparation](#application-preparation)
- [Containerization with Docker](#containerization-with-docker)
- [Deploy to Render](#deploy-to-render)
- [CI/CD with GitHub Actions](#cicd-with-github-actions)
- [CI/CD with Jenkins](#cicd-with-jenkins)
- [Resources](#resources)

---

## Application Preparation

### pom.xml Configuration

```xml
<dependencies>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
  </dependency>
  <dependency>
      <groupId>me.paulschwarz</groupId>
      <artifactId>spring-dotenv</artifactId>
      <version>3.0.0</version>
  </dependency>
</dependencies>
```

### application.properties Configuration

```properties
# Application Info
spring.profiles.active=${SPRING_PROFILES_ACTIVE:prod}

# Server Configuration
server.port=${SERVER_PORT:8080}

# Database Configuration
spring.datasource.url=${DATABASE_URL}
spring.datasource.username=${DATABASE_USERNAME}
spring.datasource.password=${DATABASE_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA Configuration
spring.jpa.show-sql=false

# Connection Pool Settings
spring.datasource.hikari.connection-timeout=30000

# Actuator (Health Check)
management.endpoints.web.exposure.include=health,info
```

### application-dev.properties Configuration

```properties
# JPA Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.format_sql=true

# Connection Pool Settings
spring.datasource.hikari.maximum-pool-size=5
spring.datasource.hikari.minimum-idle=2

# Actuator (Health Check)
management.endpoint.health.show-details=always

# Logging
logging.level.root=DEBUG
logging.level.com.fahmi.rendersupabaseapiv2=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### application-prod.properties Configuration

```properties
# JPA Configuration
spring.jpa.hibernate.ddl-auto=validate

# Connection Pool Settings
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.minimum-idle=2

# Actuator (Health Check)
management.endpoint.health.show-details=never

# Logging
logging.level.root=INFO
logging.level.com.fahmi.rendersupabaseapiv2=INFO
logging.level.org.hibernate.SQL=ERROR
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=ERROR
```

### .env Configuration

```env
# Profile Active
SPRING_PROFILE_ACTIVE=<dev/prod>

# Server Port
PORT=8080

# Database Info
DATABASE_HOST=<db_host>
DATABASE_PORT=<db_port>
DATABASE_NAME=<db_name>
DATABASE_URL=jdbc:postgresql://${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}
DATABASE_USERNAME=<db_username>
DATABASE_PASSWORD=<db_password>
```

---

## Containerization with Docker

### Dockerfile (Multi-stage Build)

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Create a non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser

COPY --from=build /app/target/*.jar app.jar

# Optional: only needed if app writes to /app
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 8080

# Add a health check: 
# - Call the `/api/health` endpoint every 30 seconds to verify if the application is running normally
# - If it fails 3 consecutive times within a 3-second timeout, the container is considered unhealthy
# - The first check will start after the container has been running for 40 seconds
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
CMD wget --no-verbose --tries=1 --spider http://localhost:8080/api/health || exit 1

# Run the application with a minimum memory allocation of 256MB and a maximum of 512MB.
ENTRYPOINT ["java", "-Xmx512m", "-Xms256m", "-jar", "app.jar"]
```

---

### .dockerignore

```
target/
.mvn/
.git/
.gitignore
*.md
.env
```

### docker-compose.yml (Development)

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "${SERVER_PORT}:8080"
    environment:
      SPRING_PROFILE_ACTIVE: ${SPRING_PROFILE_ACTIVE}
      SPRING_DATASOURCE_URL: ${DATABASE_URL}
      SPRING_DATASOURCE_USERNAME: ${DATABASE_USERNAME}
      SPRING_DATASOURCE_PASSWORD: ${DATABASE_PASSWORD}
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: ${DATABASE_NAME}
      POSTGRES_USER: ${DATABASE_USERNAME}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    ports:
      - "${DATABASE_PORT}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## Deploy to Render

### Deploy Steps

1. Push code to GitHub
2. Login to [Render.com](https://render.com)
3. New > Web Service
4. Connect repository
5. Add Environment Variables
6. Create Web Service

---

## CI/CD with GitHub Actions

### .github/workflows/keep-alive.yml (to prevent cold start in free Render)

```yaml
name: Keep-Alive

on:
  schedule:
    - cron: '*/14 * * * *' # Every 14 minutes
    # - cron: '*/14 8-18 * * 1-5' # Every 14 minutes between 8 and 18 between Monday and Friday

jobs:
  ping:
    runs-on: ubuntu-latest

    env:
      RENDER_URL: ${{ secrets.RENDER_URL }}
      HEALTH_ENDPOINT: <health_check_endpoiny>

    steps:
      - name: Ping health check endpoint
        run: curl -sS $RENDER_URL/$HEALTH_ENDPOINT > /dev/null
```


### .github/workflows/deploy.yml

```yaml
name: Build and Deploy

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: "ghcr.io/<username>/<repository>:latest"
      GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}
      RENDER_DEPLOY_HOOK: ${{ secrets.RENDER_HOOK }}

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Log in to GHCR
        run: echo "$GHCR_TOKEN" | docker login ghcr.io -u <username> --password-stdin

      - name: Build Docker image
        run: docker build -t $IMAGE_NAME .

      - name: Push Docker image to GHCR
        run: docker push $IMAGE_NAME

      - name: Trigger Render Deploy Hook
        run: curl -X POST "$RENDER_DEPLOY_HOOK"

      - name: Notify success
        if: success()
        run: echo 'Deployment successful!'

      - name: Notify failure
        if: failure()
        run: echo 'Deployment failed!'

      - name: Clean workspace
        if: always()
        run: |
          echo 'Cleaning up workspace...'
          sudo rm -rf * || true
          echo 'Done!'
```

### Required GitHub Secrets
- `GHCR_TOKEN` (from GitHub > Settings > Developer settings > Tokens)
- `RENDER_DEPLOY_HOOK` (from Render dashboard > Settings > Deploy Hook)

### Set GitHub Secrets
1. Open repository
2. Settings → Secrets and variables → Actions
3. New repository secret
4. `Name`: <secret_name>
5. `Value`: <secret_value>
6. Add secret

---

## CI/CD with Jenkins

### Jenkinsfile

```groovy
pipeline {
  agent any

  environment {
    IMAGE_NAME = "ghcr.io/<username>/<repository>:latest"
    GHCR_TOKEN = credentials('GHCR_TOKEN')
    RENDER_DEPLOY_HOOK = credentials('RENDER_HOOK')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Docker Build & Push') {
      steps {
        sh "docker build -t $IMAGE_NAME ."
        sh "echo $GHCR_TOKEN | docker login ghcr.io -u <username> --password-stdin"
        sh "docker push $IMAGE_NAME"
      }
    }

    stage('Trigger Render Deploy') {
      steps {
        sh "curl -X POST $RENDER_DEPLOY_HOOK"
      }
    }
  }

  post {
    success {
      echo 'Deployment successful!'
    }
    failure {
      echo 'Deployment failed!'
    }
    always {
      echo 'Cleaning up workspace...'
      cleanWs()
      echo 'Done!'
    }
  }
}
```

### Required Jenkins UI Credentials
- `GHCR_TOKEN` (from GitHub > Settings > Developer settings > Tokens)
- `RENDER_DEPLOY_HOOK` (from Render dashboard > Settings > Deploy Hook)

### Set Jenkins UI Credentials
1. Open Jenkins UI → Manage Jenkins → Credentials
2. System → Global
3. Add credentials
4. `Kind`: Secret text, 
5. `Secret`: <GHCR_TOKEN/RENDER_DEPLOY_HOOK>
6. `ID`: <credential_id>
7. `Description`: <credential_desc>
8. Create
---

### Jenkins Setup
1. Jenkins UI → New Item
2. `Name`: <item_name>
3. Select Pipeline
4. `Definition`: Pipeline script from SCM
5. `SCM`: Git
6. `Repository URL`: <repository_url>
7. `Script path`: Jenkinsfile
8. Save

---

## Resources

- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Docker Documentation](https://docs.docker.com/)
- [Render Documentation](https://render.com/docs)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
