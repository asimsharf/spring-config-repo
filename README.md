# Spring Cloud Config Repository

**Single Source of Truth** for all microservices configuration across all environments.

## 📁 Repository Structure

```
spring-config-repo/
├── application.yml                    # Shared configuration (all services)
├── application-local.yml              # Shared local overrides
├── application-dev.yml                # Shared dev overrides
├── application-staging.yml            # Shared staging overrides
├── application-prod.yml               # Shared prod overrides
│
├── config-server.yml                  # Config Server configuration
├── discovery-server.yml               # Eureka Server configuration
│
├── api-gateway.yml                    # API Gateway base configuration
├── api-gateway-local.yml              # API Gateway local profile
├── api-gateway-dev.yml                # API Gateway dev profile
├── api-gateway-staging.yml            # API Gateway staging profile
├── api-gateway-prod.yml               # API Gateway production profile
│
├── user-service.yml                   # User Service base configuration
├── user-service-local.yml             # User Service local profile
├── user-service-dev.yml               # User Service dev profile
├── user-service-staging.yml          # User Service staging profile
├── user-service-prod.yml              # User Service production profile
│
├── README.md                          # This file
└── CONFIG_REVIEW.md                   # Technical review document
```

## 🎯 Configuration Hierarchy

Spring Cloud Config applies configurations in this order (highest to lowest priority):

1. **Profile-Specific Service Config** (`user-service-prod.yml`)
2. **Service Base Config** (`user-service.yml`)
3. **Profile-Specific Shared Config** (`application-prod.yml`)
4. **Shared Base Config** (`application.yml`)

### Example: `user-service` with `prod` profile

```
Priority 1: user-service-prod.yml     (highest - service + profile)
Priority 2: user-service.yml           (service base)
Priority 3: application-prod.yml       (shared profile)
Priority 4: application.yml            (lowest - shared base)
```

## 🌍 Environment Profiles

### `local` - Local Development
- **Purpose:** Developer-friendly settings
- **Database:** H2 in-memory
- **Security:** Disabled (no authentication)
- **Logging:** DEBUG level
- **Actuator:** All endpoints exposed
- **Flyway:** Disabled (using H2 auto-update)
- **Tracing:** 100% sampling

### `dev` - Development Server
- **Purpose:** Development server testing
- **Database:** MySQL (local or remote)
- **Security:** Basic (can enable JWT)
- **Logging:** INFO level
- **Actuator:** Standard endpoints
- **Flyway:** Enabled
- **Tracing:** 100% sampling

### `staging` - Pre-Production
- **Purpose:** Production-like testing
- **Database:** MySQL (staging environment)
- **Security:** Full security enabled
- **Logging:** INFO level
- **Actuator:** Limited endpoints
- **Flyway:** Enabled
- **Tracing:** 50% sampling

### `prod` - Production
- **Purpose:** Production deployment
- **Database:** MySQL (production)
- **Security:** Full security, minimal exposure
- **Logging:** WARN level (minimal)
- **Actuator:** Minimal endpoints
- **Flyway:** Enabled
- **Tracing:** 10% sampling

## 🔐 Environment Variables

Configuration uses environment variables for sensitive values:

### Database
```bash
export DB_URL=jdbc:mysql://localhost:3306/usersdb
export DB_USERNAME=user
export DB_PASSWORD=password
export DB_POOL_SIZE=20
export DB_MIN_IDLE=10
```

### Service Discovery
```bash
export EUREKA_URI=http://localhost:8761/eureka
```

### Ports
```bash
export PORT=9001  # Optional, uses defaults if not set
```

### CORS
```bash
export CORS_ALLOWED_ORIGINS=http://localhost:3000,https://yourdomain.com
```

### Logging & Tracing
```bash
export LOG_LEVEL=INFO
export TRACING_SAMPLING_PROBABILITY=1.0
```

## 📝 Adding a New Service

1. **Create base configuration:**
   ```yaml
   # {service-name}.yml
   spring:
     application:
       name: {service-name}
   server:
     port: ${PORT:9002}
   ```

2. **Create profile-specific files:**
   - `{service-name}-local.yml`
   - `{service-name}-dev.yml`
   - `{service-name}-staging.yml`
   - `{service-name}-prod.yml`

3. **Add route to API Gateway:**
   ```yaml
   # api-gateway.yml
   spring:
     cloud:
       gateway:
         routes:
           - id: {service-name}
             uri: lb://{service-name}
             predicates:
               - Path=/api/v1/{resource}/**
   ```

## 🔒 Security Best Practices

### ✅ DO:
- Use environment variables: `${DB_PASSWORD}`
- Use placeholders for all secrets
- Document required environment variables
- Use external secret managers for production (Vault, K8s Secrets, AWS Secrets Manager)
- Keep production configs minimal

### ❌ DON'T:
- Commit passwords, tokens, or secrets
- Hardcode sensitive values
- Use `*` for actuator exposure in production
- Store secrets in Git (even encrypted)

### Secret Management Pattern

```yaml
# ❌ BAD
spring:
  datasource:
    password: mypassword123

# ✅ GOOD
spring:
  datasource:
    password: ${DB_PASSWORD}
    username: ${DB_USERNAME:defaultuser}
```

## 🚀 Usage

### Local Development
```bash
# Set profile
export SPRING_PROFILES_ACTIVE=local

# Start services (they will load config from Config Server)
cd /Users/asimsharf/Desktop/spring-microservices
./start-all.sh
```

### Development Server
```bash
export SPRING_PROFILES_ACTIVE=dev
export DB_URL=jdbc:mysql://localhost:3306/usersdb
export DB_USERNAME=user
export DB_PASSWORD=password
```

### Production
```bash
export SPRING_PROFILES_ACTIVE=prod
export DB_URL=${PROD_DB_URL}
export DB_USERNAME=${PROD_DB_USER}
export DB_PASSWORD=${PROD_DB_PASS}
export CORS_ALLOWED_ORIGINS=https://yourdomain.com
```

## 🔄 Config Refresh

Services can refresh configuration without restart:

```bash
# Refresh a specific service
curl -X POST http://localhost:9001/actuator/refresh

# Refresh all services (if using Spring Cloud Bus)
curl -X POST http://localhost:8888/actuator/bus-refresh
```

**Note:** Services must have `@RefreshScope` on beans that need refresh.

## 📊 Monitoring

### Config Server Endpoints
- Health: `http://localhost:8888/actuator/health`
- Environment: `http://localhost:8888/actuator/env`
- Config: `http://localhost:8888/{service}/{profile}`

### Example: Get user-service config
```bash
curl http://localhost:8888/user-service/local
```

## 🐛 Troubleshooting

### Config Not Loading
1. Check Config Server is running: `http://localhost:8888/actuator/health`
2. Verify Git repository URL in Config Server
3. Check service name matches exactly (case-sensitive)
4. Verify profile is set: `SPRING_PROFILES_ACTIVE`

### Wrong Configuration Applied
1. Check profile precedence
2. Verify environment variables are set
3. Check Config Server logs
4. Use `/actuator/env` endpoint to see resolved config

### Secrets Not Working
1. Verify environment variables are set
2. Check `${VARIABLE}` syntax is correct
3. Ensure no hardcoded values override env vars
4. Use external secret manager for production

## 📚 References

- [Spring Cloud Config Documentation](https://spring.io/projects/spring-cloud-config)
- [Configuration Best Practices](./CONFIG_REVIEW.md)
- [Microservices Architecture](../spring-microservices/ARCHITECTURE.md)

---

**Last Updated:** 2024  
**Maintained by:** Spring Boot Microservices Team

