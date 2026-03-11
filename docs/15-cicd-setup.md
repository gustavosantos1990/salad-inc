# 15 — CI/CD Pipeline Setup

This guide sets up a complete, automated CI/CD pipeline for Java/Spring Boot applications on the Salad.Inc infrastructure. When finished, every `git push` will automatically test, analyse, scan, build, and deploy your application — with no manual steps.

## Architecture Overview

```
Developer pushes code
        │
        ▼
  GitLab (repository.salad.com)          ← source of truth; triggers CI
        │
        ▼
  GitLab Runner (containerization.salad.com)   ← Docker executor
        │
        ├── Stage 1: governance
        │       salad-ci-tools image
        │       print-commit-info → commit SHA, author, branch, message
        │       print-tree → repo structure + resources folder size
        │
        ├── Stage 2: unit-test
        │       salad-ci-tools image
        │       mvn test → JUnit report archived in GitLab
        │
        ├── Stage 3: sonar
        │       mvn sonar:sonar → SonarQube (same VM)
        │       Quality Gate must pass or pipeline STOPS
        │
        ├── Stage 4: security (two parallel jobs)
        │       owasp  → mvn dependency-check:check (pom.xml CVE scan)
        │       trivy  → aquasec/trivy image (Docker image CVE scan)
        │
        └── Stage 5: build-deploy
                mvn package → docker build → docker push to Nexus
                [main only] docker stack deploy → Swarm rolling update
                [main only] health check → /actuator/health
```

### Branching strategy — GitHub Flow

All development happens in short-lived feature branches off `main`. A Merge Request (MR) triggers stages 1–3 (no deploy). Merging to `main` runs all 4 stages including deploy. Branches should not live longer than a few days to avoid merge conflicts.

### Component map

| Component | VM | Address |
|---|---|---|
| GitLab | repository.salad.com | 192.168.123.20 |
| GitLab Runner | containerization.salad.com | 192.168.123.22 |
| SonarQube | containerization.salad.com | sonarqube.salad.com |
| Nexus Docker push | artifact-repository.salad.com | registry-push.salad.com |
| Nexus Docker pull | artifact-repository.salad.com | registry.salad.com |
| Docker Swarm | containerization.salad.com | 192.168.123.22 |

---

## 1. Expand Containerization VM RAM (2 GB → 4 GB)

SonarQube requires approximately 3 GB of RAM on its own (Elasticsearch embedded inside SonarQube is the heavyweight). With the existing workloads on `containerization` (Traefik, Portainer, pgAdmin, application containers) and the upcoming GitLab Runner, the VM needs to grow from 2 GB to 4 GB before anything else is deployed.

> **Why not a separate VM?** SonarQube is an internal quality tool — it does not serve end-user traffic. Running it as a Swarm stack alongside other internal tooling is consistent with how Portainer and pgAdmin are already deployed on this VM, and avoids provisioning yet another QEMU VM on already limited host resources.

### 1.1 Check current allocation

On the host machine:

```bash
virsh dominfo containerization | grep -i memory
```

Expected: `Max memory: 2097152 kB` (2 GB).

### 1.2 Increase max and current memory

```bash
# Set the new maximum (in kibibytes: 4 GB = 4194304 KiB)
virsh setmaxmem containerization 4194304 --config

# Set the live memory (VM must be running)
virsh setmem containerization 4194304 --live --config
```

`--config` persists the change across reboots. `--live` applies it immediately without a restart.

### 1.3 Verify

```bash
virsh dominfo containerization | grep -i memory
# Max memory:     4194304 kB
# Used memory:    4194304 kB
```

Inside the VM:

```bash
free -h
# Should show ~3.8 GB total
```

---

## 2. Install GitLab Runner on the Containerization VM

The GitLab Runner is the agent that actually executes your pipeline jobs. It connects to GitLab, polls for pending jobs, pulls the specified Docker image for each job, runs the scripts inside a fresh container, and reports results back to GitLab.

We install it on `containerization.salad.com` because:
- Docker is already running there — no extra setup
- The Runner can mount the local Docker socket to deploy Swarm stacks directly
- Network latency to Nexus and SonarQube is near-zero (same subnet)

### 2.1 Install the Runner package

SSH into `containerization.salad.com`:

```bash
ssh root@containerization.salad.com
```

Add the GitLab Runner apt repository and install:

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | bash
apt-get install -y gitlab-runner
```

Enable and start the service:

```bash
systemctl enable --now gitlab-runner
```

### 2.2 Register the Runner with GitLab

The registration command links this Runner to your GitLab instance. You need an **authentication token** from GitLab first.

**Get the token from GitLab (GitLab 16+):**
1. Log in to `http://repository.salad.com`
2. Go to **Admin Area** → **CI/CD** → **Runners**
3. Click **New instance runner**
4. Select "Linux" as the platform, "Docker" as the architecture, and "Docker" as the executor.
5. Copy the **authentication token**.

**Register:**

```bash
gitlab-runner register \
  --non-interactive \
  --url "http://repository.salad.com" \
  --token "<YOUR_AUTHENTICATION_TOKEN>" \
  --executor "docker" \
  --docker-image "registry.salad.com/salad-ci-tools:latest" \
  --description "containerization-runner" \
  --docker-privileged \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-network-mode "host"
```

**Key flags explained:**
- `--executor docker` — each CI job runs inside its own fresh Docker container. The image used is the one specified in the job's `image:` field.
- `--docker-image` — the default image when a job doesn't specify one. We set it to our `salad-ci-tools` image.
- `--docker-privileged` — required for the build stage which runs Docker inside Docker (DinD) to build our application's Docker image.
- `--docker-volumes /var/run/docker.sock:/var/run/docker.sock` — mounts the host Docker socket into every job container. This is what allows the deploy stage to run `docker stack deploy` without SSH — it talks directly to the host Swarm manager.
- `--docker-network-mode host` — makes job containers share the host network, so they can reach `sonarqube.salad.com` and `registry-push.salad.com` by hostname through our dnsmasq DNS.

### 2.3 Verify the Runner config

The registration writes to `/etc/gitlab-runner/config.toml`. Check it:

```bash
cat /etc/gitlab-runner/config.toml
```

The `[runners.docker]` section should look like:

```toml
[[runners]]
  name = "containerization-runner"
  url = "http://repository.salad.com"
  token = "glrt-..."
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "registry.salad.com/salad-ci-tools:latest"
    privileged = true
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
    network_mode = "host"
    extra_hosts = ["registry-push.salad.com:192.168.123.30", "registry.salad.com:192.168.123.30"]
```

> The `extra_hosts` entries ensure the job containers can resolve our internal registry hostnames even if dnsmasq resolution has any hiccup inside the DinD environment.

### 2.4 Verify the Runner is connected

```bash
gitlab-runner verify
# Runner ... is alive
```

Also check in GitLab: **Admin Area** → **CI/CD** → **Runners** — the `containerization-runner` should appear with a green dot.

---

## 3. Deploy SonarQube on the Containerization VM

SonarQube is a static code analysis platform. It collects test coverage data and checks for code smells, bugs, and security hotspots. In our pipeline, the sonar stage submits analysis results to SonarQube and then polls its **Quality Gate** — a configurable pass/fail threshold. If the gate says "Error", the pipeline job fails and the pipeline stops: the build never runs and nothing gets deployed.

SonarQube requires:
- A PostgreSQL database (we reuse `database.salad.com`)
- `vm.max_map_count` ≥ 524288 on the host kernel (Elasticsearch requirement)
- ~3 GB RAM

### 3.1 Create the PostgreSQL database

SSH into `database.salad.com`:

```bash
ssh root@database.salad.com
sudo -u postgres psql
```

```sql
CREATE USER sonarqube WITH PASSWORD 'sonar_password';
CREATE DATABASE sonarqube OWNER sonarqube;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonarqube;
\q
```

### 3.2 Set `vm.max_map_count` on the containerization VM

SonarQube embeds Elasticsearch, which refuses to start if the kernel's virtual memory map limit is too low. Set it persistently:

```bash
# Apply immediately (no reboot needed)
sysctl -w vm.max_map_count=524288

# Persist across reboots
echo "vm.max_map_count=524288" >> /etc/sysctl.d/99-sonarqube.conf
```

### 3.3 Create the Swarm stack file

```bash
mkdir -p /opt/stacks/sonarqube
cat > /opt/stacks/sonarqube/sonarqube.yml << 'EOF'
version: '3.8'

services:
  sonarqube:
    image: sonarqube:community
    environment:
      SONAR_JDBC_URL: "jdbc:postgresql://database.salad.com:5432/sonarqube"
      SONAR_JDBC_USERNAME: "sonarqube"
      SONAR_JDBC_PASSWORD: "sonar_password"
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_extensions:/opt/sonarqube/extensions
    ports:
      - "9001:9000"    # Nginx proxies sonarqube.salad.com → containerization:9001
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 10s
      resources:
        limits:
          memory: 3g
        reservations:
          memory: 2g

volumes:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:
EOF
```

> **Why no Traefik network?** SonarQube is accessed through a dedicated Nginx block that proxies directly to `containerization.salad.com:9001` (the published host port). It does not go through Traefik — Traefik is only used for `*.app.salad.com` application services.

Deploy the stack:

```bash
docker stack deploy -c /opt/stacks/sonarqube/sonarqube.yml sonarqube
```

Monitor startup (SonarQube takes 2–3 minutes on first boot while it initialises the database schema):

```bash
docker service logs sonarqube_sonarqube --follow
# Wait for: "SonarQube is operational"
```

### 3.4 Add Nginx proxy block

On `proxy.salad.com`, add to `/etc/nginx/conf.d/services.conf`:

```nginx
# SonarQube
server {
    listen 80;
    server_name sonarqube.salad.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name sonarqube.salad.com;

    ssl_certificate     /etc/nginx/ssl/salad.com.crt;
    ssl_certificate_key /etc/nginx/ssl/salad.com.key;

    # SonarQube uploads analysis reports which can be large
    client_max_body_size 0;

    location / {
        set $backend_sonar containerization.salad.com:9001;
        proxy_pass http://$backend_sonar;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300;
    }
}
```

Reload Nginx:

```bash
nginx -t && nginx -s reload
```

### 3.5 Add DNS entry

On `dns.salad.com`:

```bash
echo "address=/sonarqube.salad.com/192.168.123.30" >> /etc/dnsmasq.conf
service dnsmasq restart
```

### 3.6 Initial SonarQube setup

Open `https://sonarqube.salad.com` in your browser.

Default credentials: **admin / admin** — you will be forced to change the password on first login.

#### Create a project token

1. Log in as `admin`
2. Go to **My Account** → **Security** → **Generate Tokens**
3. Token name: `gitlab-ci`
4. Token type: **Global Analysis Token**
5. Click **Generate** and **copy the token** immediately — you won't see it again

This token is what the pipeline uses to authenticate with SonarQube. It will be stored as a GitLab CI/CD variable (`SONAR_TOKEN`).

#### Configure the Quality Gate

SonarQube ships with a "Sonar way" default quality gate. We will customise it:

1. Go to **Quality Gates** → **Create** → Name: `salad-gate`
2. Add conditions:
   - **Coverage** on new code: **is less than 80%** → Error
   - **Reliability Rating** on new code: **is worse than A** → Error (blocks on any bug)
   - **Security Rating** on new code: **is worse than A** → Error
   - **Duplicated Lines (%)** on new code: **is greater than 5%** → Warning (not a blocker)
3. Click **Set as Default**

> The gate evaluates only **new code** introduced in the MR or push — it does not fail on pre-existing technical debt. This way teams can adopt SonarQube incrementally without being immediately blocked by legacy issues.

---

## 4. Bootstrap `salad/ci-tools` — Shared Utility Image

The `salad/ci-tools` image is a custom Docker image stored in Nexus that serves as the **default execution environment** for pipeline jobs. It extends the standard Maven/Java 25 image with small utility scripts baked in.

**Why a custom image instead of inline `apt-get install`?**
If you install tools with `apt-get` inside your job's `script:` block, that installation runs on every single pipeline execution — adding 60–120 seconds of overhead. By pre-baking tools into a Docker image, each job gets a ready-to-go environment in seconds. The image itself only needs to be rebuilt when the tools or scripts change.

**What lives in `ci-tools` (and what doesn't):**
- ✅ Git log/commit info script — generic, used by the `unit-test` stage to annotate every pipeline run
- ✅ Directory tree printer — generic repo structure snapshot logged for debugging
- ❌ OWASP, Trivy — these use their own official images, not `ci-tools`
- ❌ Application logic — `ci-tools` knows nothing about your specific app

### 4.1 Create the GitLab repo

In GitLab (`https://repository.salad.com`):
- Create a new project under the `salad` group: **salad / ci-tools**
- Make it internal visibility

Clone it to your workstation and create this structure:

```
ci-tools/
├── Dockerfile
├── scripts/
│   ├── print-commit-info.sh
│   └── print-tree.sh
└── .gitlab-ci.yml
```

### 4.2 Scripts

**`scripts/print-commit-info.sh`** — displays git metadata at the start of every job so you always know exactly what code is being processed:

```bash
#!/bin/bash
set -e

echo "========================================"
echo "  CI JOB CONTEXT"
echo "========================================"
echo "  Pipeline:   ${CI_PIPELINE_ID:-local}"
echo "  Job:        ${CI_JOB_NAME:-local}"
echo "  Branch:     ${CI_COMMIT_REF_NAME:-$(git rev-parse --abbrev-ref HEAD)}"
echo "  Commit:     $(git rev-parse HEAD)"
echo "  Short SHA:  $(git rev-parse --short HEAD)"
echo "  Author:     $(git log -1 --format='%an <%ae>')"
echo "  Date:       $(git log -1 --format='%ci')"
echo "  Message:    $(git log -1 --format='%s')"
echo "========================================"
```

**`scripts/print-tree.sh`** — prints the project directory tree, useful for verifying the workspace is checked out correctly:

```bash
#!/bin/bash
set -e

echo "========================================"
echo "  PROJECT TREE"
echo "========================================"
# Skip target/ and .git/ — they are large and uninteresting
find . -not -path './.git/*' \
       -not -path './target/*' \
       -not -path './.mvn/repository/*' \
  | sort \
  | sed 's|[^/]*/|  |g'
echo "========================================"

# Also report resources folder size specifically
if [ -d "src/main/resources" ]; then
  echo "  src/main/resources/ size: $(du -sh src/main/resources | cut -f1)"
fi
```

### 4.3 Dockerfile

```dockerfile
FROM maven:3.9.13-eclipse-temurin-25

# Install utilities used by scripts
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    tree \
    && rm -rf /var/lib/apt/lists/*

# Copy utility scripts into PATH so they can be called by name in pipeline jobs
COPY scripts/print-commit-info.sh /usr/local/bin/print-commit-info
COPY scripts/print-tree.sh        /usr/local/bin/print-tree

RUN chmod +x /usr/local/bin/print-commit-info \
             /usr/local/bin/print-tree
```

> **Why extend `maven:3.9.13-eclipse-temurin-25`?** Because the `unit-test` stage runs `mvn test` — using this as the base means the image already has Maven and the JDK 25 installed. Good image layering: no redundant runtimes.

### 4.4 Pipeline to build and publish the image

**`.gitlab-ci.yml`** (in the `salad/ci-tools` repo itself):

```yaml
stages:
  - build-image

variables:
  # Push to our hosted registry in Nexus
  IMAGE_NAME: registry-push.salad.com/salad-ci-tools

build-image:
  stage: build-image
  image: docker:27
  services:
    - name: docker:27-dind
      alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login registry-push.salad.com -u gitlab-ci -p $NEXUS_CI_PASSWORD
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker tag  $IMAGE_NAME:$CI_COMMIT_SHORT_SHA $IMAGE_NAME:latest
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_NAME:latest
  only:
    - main
```

After pushing to `main`, check Nexus (`https://nexus.salad.com`) → Repositories → `docker-hosted` — the `salad-ci-tools` image should appear.

---

## 5. Bootstrap `salad/ci-templates` — Shared Pipeline YAML

The `salad/ci-templates` repo contains the **shared job definitions** that every Spring Boot project inherits. Instead of duplicating the sonar, OWASP, and Trivy steps in every project's `.gitlab-ci.yml`, you define them once here and projects include them with a single line.

When a project uses `include:`, GitLab merges its local `.gitlab-ci.yml` with the remote template at pipeline creation time. The project can override any job by redefining it with the same name.

### 5.1 Create the GitLab repo

Create **salad / ci-templates** (internal visibility) in GitLab.

Structure:

```
ci-templates/
└── global.yml
```

### 5.2 `global.yml`

This file defines stages and the three shared jobs (unit-test, sonar, security).

```yaml
# salad/ci-templates: global.yml
# Included by all Spring Boot projects via:
#   include:
#     - project: 'salad/ci-templates'
#       ref: main
#       file: '/global.yml'

stages:
  - governance    # Audit trail: who pushed what, when
  - unit-test     # JUnit results
  - sonar         # Quality gate
  - security      # OWASP + Trivy
  - build-deploy  # Build image, push, deploy (project-defined)

# ─── Variables (projects override as needed) ──────────────────────────────────
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2"
  SONAR_HOST_URL: "https://sonarqube.salad.com"
  DOCKER_REGISTRY_PUSH: "registry-push.salad.com"
  DOCKER_REGISTRY_PULL: "registry.salad.com"

# ─── Stage 1: governance ──────────────────────────────────────────────────────
# Runs first on every pipeline — produces an audit trail showing exactly
# which commit, author, and branch triggered this run, plus the repo structure.
# Uses the salad-ci-tools custom image (Maven + JDK 25 + baked-in scripts).
governance:
  stage: governance
  image: registry.salad.com/salad-ci-tools:latest
  script:
    - print-commit-info   # Commit SHA, author, branch, message
    - print-tree          # Directory tree + resources folder size

# ─── Stage 2: unit-test ───────────────────────────────────────────────────────
unit-test:
  stage: unit-test
  # Use the official Maven/JDK image — ci-tools is governance-only (no Maven)
  image: maven:3.9.13-eclipse-temurin-25
  script:
    - mvn test -s .mvn/settings.xml
  artifacts:
    # Publish test results to GitLab's built-in test report viewer
    reports:
      junit: target/surefire-reports/TEST-*.xml
    # Keep compiled classes so sonar stage can pick up coverage data
    paths:
      - target/surefire-reports/
      - target/classes/
    expire_in: 1 hour
  cache:
    key: "$CI_PROJECT_PATH-maven"
    paths:
      - .m2/

# ─── Stage 3: sonar ───────────────────────────────────────────────────────────
sonar:
  stage: sonar
  image: maven:3.9.13-eclipse-temurin-25
  dependencies:
    # Pull the test report artifacts from the unit-test job
    - unit-test
  script:
    # sonar.qualitygate.wait=true makes Maven poll SonarQube until analysis
    # is complete, then fail the job if the Quality Gate says "Error".
    - mvn sonar:sonar
        -s .mvn/settings.xml
        -Dsonar.host.url=$SONAR_HOST_URL
        -Dsonar.token=$SONAR_TOKEN
        -Dsonar.projectKey=$SONAR_PROJECT_KEY
        -Dsonar.qualitygate.wait=true
  cache:
    key: "$CI_PROJECT_PATH-maven"
    paths:
      - .m2/
  allow_failure: false  # A failing quality gate STOPS the pipeline

# ─── Stage 4: security ────────────────────────────────────────────────────────

# Job A: OWASP Dependency-Check
# Scans pom.xml dependencies against the NVD (National Vulnerability Database).
# Archives the HTML report as a GitLab artifact for review.
owasp:
  stage: security
  image: maven:3.9.13-eclipse-temurin-25
  script:
    - mvn org.owasp:dependency-check-maven:check
        -s .mvn/settings.xml
        -DfailBuildOnCVSS=9
        -DsuppressionFile=dependency-check-suppressions.xml
        --no-transfer-progress
  artifacts:
    when: always   # Archive the report even when the job fails
    paths:
      - target/dependency-check-report.html
    expire_in: 1 week
  cache:
    key: "$CI_PROJECT_PATH-maven"
    paths:
      - .m2/
  allow_failure: true  # Set to false once baseline CVEs are suppressed

# Job B: Trivy image scan
# Scans the built Docker image for OS-layer CVEs in the JRE Alpine base.
# Must run after build-deploy pushes the image to Nexus.
trivy:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  variables:
    TRIVY_USERNAME: "gitlab-ci"
    TRIVY_PASSWORD: "$NEXUS_CI_PASSWORD"
    TRIVY_AUTH_URL: "https://registry-push.salad.com"
    TRIVY_CACHE_DIR: "$CI_PROJECT_DIR/.trivy-cache"
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL
        --no-progress
        registry.salad.com/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA
  cache:
    key: trivy-db
    paths:
      - .trivy-cache/
  needs:
    - build-deploy   # Image must exist before Trivy can scan it
  allow_failure: true  # Set to false when baseline is established
```

> **`DfailBuildOnCVSS=9`** — CVSS score 9.0+ means "Critical". Scores 7.0–8.9 are "High" and will still appear in the report but won't fail the job. Adjust to `7` when your team is ready to tackle High-severity dependencies too.

> **`allow_failure: true` on security jobs** — intentionally lenient at first. When you adopt this pipeline in a codebase that has never been scanned, you will almost certainly find existing CVEs. Setting `allow_failure: false` immediately would block every deployment. Review the first run's reports, suppress known false positives or accepted risks in `dependency-check-suppressions.xml`, and then flip to `false`.

---

## 6. Add `.gitlab-ci.yml` to Your Spring Boot Project

With the shared template bootstrapped, each project's own pipeline file only needs to:
1. Include the shared template
2. Define the project-specific `build-deploy` job (image name, stack name, service name)
3. Provide the Maven settings file for Nexus

### 6.1 Maven settings for Nexus

Create `.mvn/settings.xml` in the project root. This routes all Maven dependency downloads through your Nexus instance instead of hitting Maven Central directly:

```xml
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-releases</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://artifact-repository.salad.com:8081/repository/maven-group/</url>
    </mirror>
  </mirrors>
</settings>
```

### 6.2 Swarm stack file

Your project should have a `swarm-stack.yml` (as described in doc 14). The `build-deploy` job references this file by name. Example for the `demo` project:

```yaml
# swarm-stack.yml
version: '3.8'

services:
  demo:
    image: registry.salad.com/demo:latest
    networks:
      - traefik-public
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.demo.rule=Host(`demo.app.salad.com`)"
        - "traefik.http.routers.demo.entrypoints=web"
        - "traefik.http.services.demo.loadbalancer.server.port=8080"
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

networks:
  traefik-public:
    external: true
```

### 6.3 `.gitlab-ci.yml`

```yaml
# demo - .gitlab-ci.yml

include:
  - project: 'salad/ci-templates'
    ref: master
    file: '/global.yml'

# Project-specific variables
variables:
  # Used by the sonar stage (from global.yml)
  SONAR_PROJECT_KEY: "demo"
  # Name of the Docker image (without registry prefix)
  APP_IMAGE_NAME: "demo"
  # Name of the Docker Swarm stack
  SWARM_STACK_NAME: "demo"
  # Nexus credentials passed down to all Maven jobs
  NEXUS_USER: "gitlab-ci"
  NEXUS_PASSWORD: "$NEXUS_CI_PASSWORD"

# ─── Stage 5: build ──────────────────────────────────────────────────────────
# Runs on ALL branches — image is available for Trivy to scan on any branch.
build:
  stage: build
  image: registry.salad.com/docker:27
  variables:
    DOCKER_HOST: unix:///var/run/docker.sock
    IMAGE_PUSH: "registry-push.salad.com/$APP_IMAGE_NAME"
  before_script:
    - docker login registry-push.salad.com -u gitlab-ci -p $NEXUS_CI_PASSWORD
  script:
    - docker build -t $IMAGE_PUSH:$CI_COMMIT_SHORT_SHA -t $IMAGE_PUSH:latest .
    - docker push $IMAGE_PUSH:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_PUSH:latest

# ─── Stage 7: deploy ─────────────────────────────────────────────────────────
# Runs on master ONLY, after Trivy (post-build) has scanned the image.
deploy:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "master"'
  image: registry.salad.com/docker:27
  variables:
    DOCKER_HOST: unix:///var/run/docker.sock
    IMAGE_PULL: "registry.salad.com/$APP_IMAGE_NAME"
  before_script:
    - apk add --no-cache curl
    - docker login registry.salad.com -u gitlab-ci -p $NEXUS_CI_PASSWORD
  script:
    - 'echo "Deploying stack: $SWARM_STACK_NAME"'
    - docker stack deploy --with-registry-auth -c swarm-stack.yml $SWARM_STACK_NAME
    - |
      for i in $(seq 1 12); do
        STATUS=$(docker service ls \
          --filter name=${SWARM_STACK_NAME}_${APP_IMAGE_NAME} \
          --format "{{.Replicas}}")
        echo "  Replicas: $STATUS"
        if echo "$STATUS" | grep -q "^2/2"; then
          echo "Deployment is healthy."
          break
        fi
        if [ "$i" -eq 12 ]; then
          echo "ERROR: Service did not become healthy in time."
          exit 1
        fi
        sleep 10
      done
```

### 6.4 Set CI/CD variables in GitLab

In GitLab, go to your project → **Settings** → **CI/CD** → **Variables** and add:

| Variable | Value | Protected | Masked |
|---|---|---|---|
| `NEXUS_CI_PASSWORD` | nexus `gitlab-ci` user password | ✅ | ✅ |
| `SONAR_TOKEN` | token generated in step 3.6 | ✅ | ✅ |
| `SONAR_HOST_URL` | `https://sonarqube.salad.com` | ❌ | ❌ |
| `SONAR_PROJECT_KEY` | `demo` | ❌ | ❌ |

> **Protected variables** are only available to pipelines running on protected branches (like `main`). **Masked variables** are redacted from job logs so credentials don't appear in output.

---

## 7. Add OWASP Plugin to `pom.xml`

The OWASP Dependency-Check Maven plugin must be declared in your project's `pom.xml` for the `owasp` job in `global.yml` to work. Add it to the `<plugins>` section:

```xml
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>10.0.3</version>
  <configuration>
    <!-- Fail only on CVSS 9+ (Critical). Adjust as your baseline improves. -->
    <failBuildOnCVSS>9</failBuildOnCVSS>
    <!-- Output HTML report to target/ -->
    <format>HTML</format>
    <outputDirectory>${project.build.directory}</outputDirectory>
    <!-- Skip test-scoped dependencies (e.g., JUnit) — they don't ship to prod -->
    <skipTestScope>true</skipTestScope>
  </configuration>
</plugin>
```

---

## 8. Prometheus & Grafana Integration

SonarQube exposes a Prometheus-compatible metrics endpoint at `/api/monitoring/metrics` (requires authentication). Add a scrape job to the Monitor VM.

### 8.1 Add SonarQube scrape to Prometheus

On `monitor.salad.com`, edit `/etc/prometheus/prometheus.yml` and add:

```yaml
  - job_name: "sonarqube"
    metrics_path: /api/monitoring/metrics
    basic_auth:
      username: "admin"
      password: "<sonarqube-admin-password>"
    scheme: https
    tls_config:
      insecure_skip_verify: true   # our self-signed cert
    static_configs:
      - targets: ["sonarqube.salad.com"]
```

Reload Prometheus:

```bash
systemctl reload prometheus
```

### 8.2 Import Grafana dashboard

In Grafana (`https://grafana.salad.com`):
1. Go to **Dashboards** → **Import**
2. Enter ID **`14681`** → **Load**
3. Select the Prometheus data source
4. Click **Import**

This dashboard shows SonarQube project metrics (issues, coverage trends, quality gate history) in Grafana alongside your VM infrastructure metrics.

---

## 9. Verification

### 9.1 SonarQube is up

```bash
curl -sk https://sonarqube.salad.com/api/system/status | python3 -m json.tool
# { "id": "...", "version": "...", "status": "UP" }
```

### 9.2 Runner is connected

```bash
gitlab-runner verify
# Runner ... is alive
```

In GitLab UI: **Admin Area** → **CI/CD** → **Runners** → green dot next to `containerization-runner`.

### 9.3 Feature branch pipeline (stages 1–3)

Push any commit to a feature branch. GitLab should immediately queue a pipeline. Verify each stage:

- **unit-test**: the job log starts with the commit info block and tree, then shows `BUILD SUCCESS`
- **sonar**: ends with `Quality Gate status: OK` (assuming tests pass coverage threshold)
- **owasp**: produces `target/dependency-check-report.html` — download it from GitLab artifacts
- **trivy**: note that Trivy runs after `build-deploy` via `needs:` — on MR pipelines it will be skipped

### 9.4 Main branch pipeline (all 4 stages + deploy)

Merge the MR. The pipeline runs all stages. Check post-deploy:

```bash
# On containerization VM — Swarm service health
docker stack services demo

# Output:
# ID      NAME                 MODE         REPLICAS   IMAGE
# xxxx    demo_demo    replicated   1/1        registry.salad.com/demo:latest

# Application health endpoint
curl -sk https://demo.app.salad.com/actuator/health
# {"status":"UP"}
```

---

## 10. Troubleshooting

### Runner not picking up jobs

```bash
# Check runner service
systemctl status gitlab-runner

# Check connectivity to GitLab
gitlab-runner verify

# Check runner logs
journalctl -u gitlab-runner -f
```

### SonarQube won't start

```bash
# Check service logs
docker service logs sonarqube_sonarqube

# Common cause 1: vm.max_map_count too low
sysctl vm.max_map_count   # must be >= 524288

# Common cause 2: Can't reach PostgreSQL
# Test from containerization VM:
psql -h database.salad.com -U sonarqube -d sonarqube -c "SELECT 1;"
```

### Quality Gate always "Error"

If the gate fails because coverage is 0%, that usually means JaCoCo is not configured. Add JaCoCo to `pom.xml`:

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals><goal>report</goal></goals>
    </execution>
  </executions>
</plugin>
```

Then add to the sonar stage's script:

```
-Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

### Docker build fails — "Cannot connect to Docker daemon"

The DinD service must be running in the job. Check that the Runner is configured with `privileged = true` in `config.toml`. Without privileged mode, DinD cannot start.

### `docker stack deploy` fails — "service not found" or "no such image"

```bash
# Check that the image was pushed to Nexus
curl -sk -u gitlab-ci:<password> \
  https://nexus.salad.com/service/rest/v1/components?repository=docker-hosted

# Check Swarm can pull from Nexus
docker pull registry.salad.com/demo:latest
```

---

## 11. k3s Migration Guide

Docker Swarm is our current orchestrator. This section documents when and how to migrate to **k3s** (lightweight Kubernetes) if that becomes the right decision.

### When to consider migrating

| Signal | Meaning |
|---|---|
| You need **namespace isolation** (staging vs production on same node) | Swarm has no namespaces |
| You want **Helm charts** for infra (Prometheus, Grafana, SonarQube in one command) | Helm doesn't work with Swarm |
| You want **GitOps** with ArgoCD or Flux | These are Kubernetes-only |
| You need **RBAC** on who can deploy which services | Kubernetes RBAC is robust; Swarm has none |
| Your team will work on **cloud platforms** (EKS, GKE, AKS) | K8s manifests are transferable; Swarm stacks are not |

### What k3s brings out of the box

k3s is a production-grade Kubernetes distribution packaged as a single ~100 MB binary. It ships with:
- **Traefik** as the default ingress controller — same routing model as our current setup
- **SQLite** as the cluster state store (vs etcd in full k8s) — much lighter for single-node
- **Flannel** for pod networking
- **Local Path Provisioner** for persistent volumes on the host filesystem
- **Helm controller** — deploy Helm charts via GitOps

### What changes in the CI/CD pipeline

The biggest change is the **Docker build story**. k3s uses `containerd` as its container runtime, not Docker. **`docker build` does not exist on a k3s node.**

| Concern | Current (Swarm) | k3s replacement |
|---|---|---|
| Build Docker images | `docker build` on Runner | **Kaniko** — builds OCI images inside a k8s pod without a daemon |
| Push to registry | `docker push` | Kaniko handles push natively |
| Deploy | `docker stack deploy` | `kubectl apply -f k8s/` |
| Stack files | `swarm-stack.yml` | `k8s/deployment.yaml` + `k8s/service.yaml` + `k8s/ingress.yaml` |
| Traefik routing | Swarm labels on services | `IngressRoute` CRD or standard `Ingress` resources |
| Portainer | Docker Swarm mode | Portainer CE supports Kubernetes (different UI) |

### Migration scope estimate

| Task | Effort |
|---|---|
| Install k3s, remove Docker Swarm | 0.5 day |
| Rewrite `swarm-stack.yml` → k8s manifests per app | 1–2 hours per app |
| Switch Runner from DinD to Kaniko | 1 day (new config, test pipeline) |
| Re-deploy Prometheus, Grafana, Loki via Helm | 0.5 day |
| Re-deploy SonarQube via Helm | 2 hours |
| Update Nginx/Traefik routes | 1 day |

Total: approximately **3–4 days** of focused work for the current infrastructure.

### Migration is NOT urgent

The Swarm pipeline we have now will handle the current workload well. The deploy stage in `.gitlab-ci.yml` is intentionally isolated to make swapping `docker stack deploy` for `kubectl apply` a one-section change when the time comes.
