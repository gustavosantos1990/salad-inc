# 13 - Deploying a Java/Spring Boot Application

This guide covers how to package a Java/Spring Boot REST API as a Docker image and deploy it to the **containerization VM** (`containerization.salad.local`) using Docker Swarm, with Traefik handling internal routing and Nginx providing SSL termination.

## Architecture Overview

```
User Browser
    ↓ https://demo-api.app.salad.local
Nginx (proxy.salad.local:443)     ← SSL termination, wildcard *.app.salad.local
    ↓ http://containerization.salad.local:80
Traefik (containerization.salad.local)  ← routes by Host header
    ↓ http://container:8080
Spring Boot App (Docker Swarm container)
```

**DNS resolution:** `*.app.salad.local` → `192.168.123.30` (Nginx) is already configured in dnsmasq. No DNS change is needed for new apps — just pick a unique subdomain.

---

## 1. Project Structure

A typical Spring Boot app should have this layout:

```
my-api/
├── src/
│   └── main/
│       ├── java/com/salad/demo/
│       │   └── DemoApplication.java
│       └── resources/
│           └── application.properties
├── Dockerfile
├── docker-compose.yml       ← for local dev (optional)
├── swarm-stack.yml          ← for production deployment
└── pom.xml
```

---

## 2. Dockerfile

Create a `Dockerfile` at the project root. We use a **multi-stage build** to keep the final image small:

```dockerfile
# Stage 1: Build
FROM maven:3.9.6-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
# Download dependencies first (cached layer)
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder /app/target/*.jar app.jar
RUN chown appuser:appgroup app.jar

USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 3. Application Configuration

In `src/main/resources/application.properties`:

```properties
server.port=8080
spring.application.name=demo-api

# Actuator health endpoint (used by Traefik for health checks)
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=always
```

---

## 4. Build and Push the Docker Image

### 4.1. Build the Image

From the project root on your **host machine** or the **containerization VM**:

```bash
docker build -t demo-api:1.0.0 .
```

### 4.2. Tag and Push to Artifactory

Artifactory acts as our Docker registry at `artifact-repository.salad.local`.

Log in to the registry:
```bash
docker login artifact-repository.salad.local:8083
```
*Use your Artifactory credentials (admin or dedicated CI user).*

Tag the image:
```bash
docker tag demo-api:1.0.0 artifact-repository.salad.local:8083/docker-local/demo-api:1.0.0
```

Push:
```bash
docker push artifact-repository.salad.local:8083/docker-local/demo-api:1.0.0
```

---

## 5. Swarm Stack File

Create `swarm-stack.yml` in your project. This is the production deployment descriptor.

The **Traefik labels** on the service are the key — they tell Traefik what hostname to route to this container.

```yaml
version: '3.8'

services:
  demo-api:
    image: artifact-repository.salad.local:8083/docker-local/demo-api:1.0.0
    networks:
      - traefik-public
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
      labels:
        # Enable Traefik for this service
        - "traefik.enable=true"

        # Router: match requests by hostname
        # Change 'demo-api' to your app name, keep the .app.salad.local suffix
        - "traefik.http.routers.demo-api.rule=Host(`demo-api.app.salad.local`)"

        # Entrypoint: Traefik listens on 'web' (port 80, HTTP from Nginx)
        - "traefik.http.routers.demo-api.entrypoints=web"

        # Service: tell Traefik which port your app listens on
        - "traefik.http.services.demo-api.loadbalancer.server.port=8080"

        # Optional: health check path (Spring Boot Actuator)
        - "traefik.http.services.demo-api.loadbalancer.healthcheck.path=/actuator/health"
        - "traefik.http.services.demo-api.loadbalancer.healthcheck.interval=10s"

networks:
  traefik-public:
    external: true
```

> **Important:** Traefik labels must be under `deploy.labels` in Swarm mode, not under `labels` at the service level. Labels at the service level are ignored by Traefik in Swarm mode.

---

## 6. Deploy to Docker Swarm

SSH into the **containerization VM**:

```bash
ssh root@containerization.salad.local
```

Pull the image (ensures the latest version is available):
```bash
docker pull artifact-repository.salad.local:8083/docker-local/demo-api:1.0.0
```

Deploy the stack:
```bash
docker stack deploy -c swarm-stack.yml demo-api
```

Check that the service is running:
```bash
docker stack services demo-api
```

Expected output:
```
ID             NAME               MODE         REPLICAS   IMAGE
xxxxxxxxxxxx   demo-api_demo-api  replicated   1/1        artifact-repository.salad.local:8083/docker-local/demo-api:1.0.0
```

`1/1` means 1 replica is running out of 1 desired.

---

## 7. Verification

### 7.1. Check Traefik Registered the Route

Traefik's dashboard is available at `https://traefik.salad.local`:

1. Go to **HTTP → Routers** — you should see `demo-api` listed.
2. Go to **HTTP → Services** — you should see `demo-api` with a healthy backend.

Or query the Traefik API directly from the containerization VM:
```bash
curl -s http://localhost:8080/api/http/routers | python3 -m json.tool | grep demo-api
```

### 7.2. Test the Application

From your host machine:
```bash
curl -k https://demo-api.app.salad.local/actuator/health
```

Expected:
```json
{"status":"UP"}
```

Without the `-k` flag (if the SSL certificate is trusted system-wide):
```bash
curl https://demo-api.app.salad.local/actuator/health
```

### 7.3. Check Container Logs

```bash
docker service logs demo-api_demo-api --follow
```

---

## 8. Updating the Application

To deploy a new version:

```bash
# Build and push the new image
docker build -t demo-api:1.0.1 .
docker tag demo-api:1.0.1 artifact-repository.salad.local:8083/docker-local/demo-api:1.0.1
docker push artifact-repository.salad.local:8083/docker-local/demo-api:1.0.1
```

Update the stack file image tag and redeploy:
```bash
# On containerization VM:
docker stack deploy -c swarm-stack.yml demo-api
```

Swarm performs a **rolling update** — the old container stays up until the new one is healthy.

---

## 9. Deploying Multiple Apps

Each new app just needs:
1. A **unique subdomain** under `*.app.salad.local` (e.g. `billing-api.app.salad.local`)
2. A **unique Traefik router name** in the labels (e.g. `demo-api` → `billing-api`)
3. Its own **stack file**

No DNS changes, no Nginx changes, no Traefik config changes — Traefik discovers new services automatically via Docker labels.

**Example for a second app (`billing-api`):**

```yaml
deploy:
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.billing-api.rule=Host(`billing-api.app.salad.local`)"
    - "traefik.http.routers.billing-api.entrypoints=web"
    - "traefik.http.services.billing-api.loadbalancer.server.port=8080"
```

---

## 10. Troubleshooting

### Container won't start
```bash
docker service ps demo-api_demo-api --no-trunc
```
Look for the error in the `ERROR` column.

### App is unreachable (502 Bad Gateway from Nginx)
```bash
# Is Traefik running?
docker service ls | grep traefik

# Is the app's service running?
docker stack services demo-api

# Can Traefik reach the container?
docker service logs traefik_traefik | tail -20
```

### Traefik shows the route but returns 503
The container is registered but failing health checks. Check app logs:
```bash
docker service logs demo-api_demo-api
```

### Image pull fails (registry auth)
```bash
# Log in to Artifactory on the containerization VM
docker login artifact-repository.salad.local:8083
```
