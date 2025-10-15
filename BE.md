# My Steps in Deploying Backend (Spring Boot) with PostgreSQL (Supabase) to Render

## Table of Contents
- [Application Preparation](#application-preparation)

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
server.port=${PORT:8080}

# Database Configuration
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
# Database Configuration (Local PostgreSQL)
spring.datasource.url=${DATABASE_URL_DEV}
spring.datasource.username=${DATABASE_USERNAME_DEV}
spring.datasource.password=${DATABASE_PASSWORD_DEV}

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
# Database Configuration (Supabase PostgreSQL)
spring.datasource.url=${DATABASE_URL_PROD}
spring.datasource.username=${DATABASE_USERNAME_PROD}
spring.datasource.password=${DATABASE_PASSWORD_PROD}

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

# Database Info (Local PostgreSQL)
DATABASE_URL_DEV=jdbc:postgresql://<db_host>:<db_port>/<db_name>
DATABASE_USERNAME_DEV=<db_username>
DATABASE_PASSWORD_DEV=<db_password>

# Database Info (Supabase PostgreSQL)
DATABASE_URL_PROD=jdbc:postgresql://<db_host>:<db_port>/<db_name>?sslmode=require
DATABASE_USERNAME_PROD=<db_username>
DATABASE_PASSWORD_PROD=<db_password>
```
