# 13 - Deploying a Java/Spring Boot Application (Manual)

> **Automated CI/CD available:** For a fully automated pipeline (build → test → scan → deploy on every `git push`), see **[15 — CI/CD Pipeline Setup](./15-cicd-setup.md)**. This guide covers the manual process for reference and local development.

This guide covers how to package a Java/Spring Boot REST API as a Docker image and deploy it to the **containerization VM** (`containerization.salad.com`) using Docker Swarm, with Traefik handling internal routing and Nginx providing SSL termination.

## Architecture Overview

```
User Browser
    ↓ https://demo.app.salad.com
Nginx (proxy.salad.com:443)       ← SSL termination, wildcard *.app.salad.com
    ↓ http://containerization.salad.com:8090
Traefik (containerization.salad.com)  ← routes by Host header
    ↓ http://container:8080
Spring Boot App (Docker Swarm container)
```

**DNS resolution:** `*.app.salad.com` → `192.168.123.30` (Nginx) is already configured in dnsmasq. No DNS change is needed for new apps — just pick a unique subdomain.

### API Gateway Pattern & CORS Resolution

To natively resolve CORS issues between frontend and backend applications, we employ an **API Gateway pattern** at the Traefik layer. 

By serving both the frontend (`/`) and the backend (`/api`) under the **exact same domain** (e.g., `demo.app.salad.com`), the browser treats all requests as Same-Origin, completely bypassing the need for complex CORS configurations in Spring Boot or React.

*   **Nginx (Edge):** Remains a simple reverse proxy passing all `*.app.salad.com` traffic to Traefik.
*   **Traefik (Ingress):** Dynamically uses Swarm labels to route `/api` traffic to the backend services (while stripping the `/api` prefix), and falls back to routing all other traffic (`/`) on that domain to the frontend.

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
spring.application.name=demo

# Actuator health endpoint (used by Traefik for health checks)
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=always
```

---

## 4. Build and Push the Docker Image

### 4.1. Build the Image

From the project root on your **host machine** or the **containerization VM**:

```bash
docker build -t demo:1.0.0 .

> **Note:** The multi-stage Dockerfile uses `maven:3.9.13-eclipse-temurin-25` as the builder and `eclipse-temurin:25-jre-alpine` as the runtime — Java 25 LTS. See the Dockerfile in the project root.
```

### 4.2. Tag and Push to Nexus

Nexus acts as our Docker registry. Push goes through `registry-push.salad.com` (hosted repo) and pulls come from `registry.salad.com` (group repo, which also proxies Docker Hub).

Log in to the push registry:
```bash
docker login registry-push.salad.com
```
*Use your Nexus credentials (`gitlab-ci` user or admin).*

Tag the image:
```bash
docker tag demo:1.0.0 registry-push.salad.com/demo:1.0.0
```

Push:
```bash
docker push registry-push.salad.com/demo:1.0.0
```

---

## 5. Swarm Stack File

Create `swarm-stack.yml` in your project. This is the production deployment descriptor.

The **Traefik labels** on the service are the key — they tell Traefik what hostname to route to this container.

```yaml
version: '3.8'

services:
  demo:
    image: registry.salad.com/demo:1.0.0
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
        # Change 'demo' to your app name, keep the .app.salad.com suffix
        - "traefik.http.routers.demo.rule=Host(`demo.app.salad.com`)"

        # Entrypoint: Traefik listens on 'web' (port 80, HTTP from Nginx)
        - "traefik.http.routers.demo.entrypoints=web"

        # Service: tell Traefik which port your app listens on
        - "traefik.http.services.demo.loadbalancer.server.port=8080"

        # Optional: health check path (Spring Boot Actuator)
        - "traefik.http.services.demo.loadbalancer.healthcheck.path=/actuator/health"
        - "traefik.http.services.demo.loadbalancer.healthcheck.interval=10s"

networks:
  traefik-public:
    external: true
```

> **Important:** Traefik labels must be under `deploy.labels` in Swarm mode, not under `labels` at the service level. Labels at the service level are ignored by Traefik in Swarm mode.

### 5.1 API Routing (Backend) vs Frontend Routing

When deploying a backend API that works alongside a frontend, use the **PathPrefix** rule and a **StripPrefix** middleware so that it serves requests on the same domain without actually requiring your Spring app controllers to map to `/api`.

**Backend Service Labels (e.g., Spring Boot):**
```yaml
      labels:
        - "traefik.enable=true"
        # Match the shared domain AND the /api path
        - "traefik.http.routers.demo.rule=Host(`demo.app.salad.com`) && PathPrefix(`/api`)"
        - "traefik.http.routers.demo.entrypoints=web"
        
        # Strip the /api prefix before it reaches Spring Boot
        - "traefik.http.middlewares.demo-strip-api.stripprefix.prefixes=/api"
        - "traefik.http.routers.demo.middlewares=demo-strip-api"
        
        - "traefik.http.services.demo.loadbalancer.server.port=8080"
```

**Frontend Service Labels (e.g., React):**
```yaml
      labels:
        - "traefik.enable=true"
        # Match the shared domain (catch-all for the UI)
        - "traefik.http.routers.demo-react.rule=Host(`demo.app.salad.com`)"
        - "traefik.http.routers.demo-react.entrypoints=web"
        - "traefik.http.services.demo-react.loadbalancer.server.port=80"
```

---

## 6. Deploy to Docker Swarm

SSH into the **containerization VM**:

```bash
ssh root@containerization.salad.com
```

Pull the image (ensures the latest version is available):
```bash
docker pull registry.salad.com/demo:1.0.0
```

Deploy the stack:
```bash
docker stack deploy -c swarm-stack.yml demo
```

Check that the service is running:
```bash
docker stack services demo
```

Expected output:
```
ID             NAME               MODE         REPLICAS   IMAGE
xxxxxxxxxxxx   demo_demo  replicated   1/1        registry.salad.com/demo:1.0.0
```

`1/1` means 1 replica is running out of 1 desired.

---

## 7. Verification

### 7.1. Check Traefik Registered the Route

Traefik's dashboard is available at `https://traefik.salad.com`:

1. Go to **HTTP → Routers** — you should see `demo` listed.
2. Go to **HTTP → Services** — you should see `demo` with a healthy backend.

Or query the Traefik API directly from the containerization VM:
```bash
curl -s http://localhost:8080/api/http/routers | python3 -m json.tool | grep demo
```

### 7.2. Test the Application

From your host machine:
```bash
curl -k https://demo.app.salad.com/actuator/health
```

Expected:
```json
{"status":"UP"}
```

Without the `-k` flag (if the SSL certificate is trusted system-wide):
```bash
curl https://demo.app.salad.com/actuator/health
```

### 7.3. Check Container Logs

```bash
docker service logs demo_demo --follow
```

---

## 8. Updating the Application

To deploy a new version:

```bash
# Build and push the new image
docker build -t demo:1.0.1 .
docker tag demo:1.0.1 registry-push.salad.com/demo:1.0.1
docker push registry-push.salad.com/demo:1.0.1
```

Update the stack file image tag and redeploy:
```bash
# On containerization VM:
docker stack deploy -c swarm-stack.yml demo
```

Swarm performs a **rolling update** — the old container stays up until the new one is healthy.

---

## 9. Deploying Multiple Apps

Each new app just needs:
1. A **unique subdomain** under `*.app.salad.com` (e.g. `billing-api.app.salad.com`)
2. A **unique Traefik router name** in the labels (e.g. `demo` → `billing-api`)
3. Its own **stack file**

No DNS changes, no Nginx changes, no Traefik config changes — Traefik discovers new services automatically via Docker labels.

**Example for a second app (`billing-api`):**

```yaml
deploy:
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.billing-api.rule=Host(`billing-api.app.salad.com`)"
    - "traefik.http.routers.billing-api.entrypoints=web"
    - "traefik.http.services.billing-api.loadbalancer.server.port=8080"
```

---

## 10. Troubleshooting

### Container won't start
```bash
docker service ps demo_demo --no-trunc
```
Look for the error in the `ERROR` column.

### App is unreachable (502 Bad Gateway from Nginx)
```bash
# Is Traefik running?
docker service ls | grep traefik

# Is the app's service running?
docker stack services demo

# Can Traefik reach the container?
docker service logs traefik_traefik | tail -20
```

### Traefik shows the route but returns 503
The container is registered but failing health checks. Check app logs:
```bash
docker service logs demo_demo
```

### Image pull fails (registry auth)
```bash
# Log in to Nexus on the containerization VM
docker login registry.salad.com
```
