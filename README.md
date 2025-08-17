# Spring PetClinic — CI/CD with GitHub Actions, JFrog (Artifactory + Xray) & Docker

This repository demonstrates how to:
- Fork and work on **Spring PetClinic**
- Build & test with **GitHub Actions**
- Package as a **Docker image**
- Push to **JFrog Artifactory** Docker registry
- **Scan** the image with **JFrog Xray** and publish a JSON report

---

## 🔰 Quick Start

### Run locally (no Docker)
```bash
mvn package
java -jar target/spring-petclinic-*.jar
# Open http://localhost:8080
```

### Build & run with Docker (locally)
```bash
mvn package
docker build -t spring-petclinic .
docker run --rm -p 8080:8080 spring-petclinic
# Open http://localhost:8080
```

### Pull & run the image from JFrog
```bash
docker pull trialgh4oxk.jfrog.io/docker-local/spring-petclinic:latest
docker run --rm -p 8080:8080 trialgh4oxk.jfrog.io/docker-local/spring-petclinic:latest
# Open http://localhost:8080
```

---

## ✅ Prerequisites

- **Git** installed (`git --version`)
- **Java 17** + **Maven** for local builds
- **Docker** Desktop/Engine
- **GitHub account** with **Actions** enabled on your fork
- **JFrog SaaS** account (Artifactory + Xray)
  - A **Docker repo** (e.g., `docker-local`)
  - **Access token** (to use as password)

---

## 🗂️ Repository Structure (key files)

```
.
├─ pom.xml
├─ src/...
├─ Dockerfile
├─ .github/
│  └─ workflows/
│     └─ build.yml
└─ reports/ (created by workflow for Xray JSON)
```

---

## 📦 Phase 1 — Preparation & Repo Setup

1. **Install & verify Git**
   ```powershell
   git --version
   ```
2. **Fork** the official Spring PetClinic repo into **your** GitHub account.
3. **Clone your fork** and enter it:
   ```powershell
   git clone https://github.com/<your-username>/spring-petclinic.git
   cd spring-petclinic
   ```
4. Confirm you see `pom.xml` and the Java sources.

✅ At this stage, you own the repo and can add workflows, Dockerfile, and docs.

---

## 🧪 Phase 2 — Build & Test Pipeline (GitHub Actions)

Create the workflow folder and file:

**PowerShell (Windows)**
```powershell
New-Item -ItemType Directory -Force .github/workflows | Out-Null
```

**macOS/Linux**
```bash
mkdir -p .github/workflows
```

Create `.github/workflows/build.yml` with:
```yaml
name: Build & Test

name: Build, Push to JFrog, and Xray Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch: # Allows manual run from GitHub UI

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # Step 1 - Checkout the repository
      - name: Checkout code
        uses: actions/checkout@v3

      # Step 2 - Set up Java 21
      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      # Step 3 - Compile the code
      - name: Compile
        run: mvn compile

      # Step 4 - Run tests
      - name: Run tests
        run: mvn test
```

Commit & push:
```powershell
git add .
git commit -m "Add build & test workflow"
git push origin main
```

Check the **Actions** tab in GitHub to see the workflow run.

✅ After this, you’ve proven the app compiles and tests pass.

---

## 🐳 Phase 3 — JFrog Setup & Docker Image Packaging

1. In **JFrog**, create a **Docker repository** (e.g., `petclinic-docker-local`).  
2. In your GitHub repo **Settings → Secrets and variables → Actions → New repository secret**, add:
   - `JFROG_USERNAME`  → your JFrog username/email
   - `JFROG_PASSWORD`  → your **access token**
   - `JFROG_URL`       → **trialgh4oxk.jfrog.io**

3. Add a `Dockerfile` in the repo root:
```dockerfile
# Use an official Maven image to build the application
FROM maven:3.9.4-eclipse-temurin-21 AS build

# Set the working directory inside the container
WORKDIR /app

# Copy the pom.xml and project files to the container
COPY pom.xml .
COPY src ./src

# Build the application
RUN mvn clean package -DskipTests

# Use an official OpenJDK image to run the application
FROM eclipse-temurin:21-jdk-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy the built JAR file from the build stage
COPY --from=build /app/target/*.jar app.jar

# Expose the default Spring Boot port
EXPOSE 8080

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

4. Extend the workflow to build & push the image (append steps **after** tests):
```yaml
      - name: Package JAR
        run: mvn clean package -DskipTests=true

      - name: Log in to JFrog Docker Registry
        run: echo "${{ secrets.JFROG_PASSWORD }}" | docker login ${{ secrets.JFROG_REGISTRY }} -u "${{ secrets.JFROG_USERNAME }}" --password-stdin

      - name: Build Docker image
        run: docker build -t ${{ secrets.JFROG_REGISTRY }}/${{ secrets.JFROG_REPO }}/spring-petclinic:latest .

      - name: Push Docker image
        run: docker push ${{ secrets.JFROG_REGISTRY }}/${{ secrets.JFROG_REPO }}/spring-petclinic:latest
```

✅ Now your pipeline builds and pushes images to JFrog at **trialgh4oxk.jfrog.io**.

---

## 🛡️ Phase 4 — Xray Security Scan (export JSON)

Add steps to **install JFrog CLI**, **configure**, **scan**, and **upload** results:

```yaml
      - name: Install JFrog CLI
        run: curl -fL https://getcli.jfrog.io | sh && sudo mv jfrog /usr/local/bin/

      - name: Configure JFrog CLI
        run: jfrog config add my-server --url=https://${ secrets.JFROG_URL } --user=${ secrets.JFROG_USERNAME } --password=${ secrets.JFROG_PASSWORD } --interactive=false

      - name: Run Xray Scan
        run: |
          mkdir -p reports
          jf docker scan ${ secrets.JFROG_URL }/petclinic-docker-local/spring-petclinic:latest --format=json > reports/xray-scan.json

      - name: Upload Scan Results
        uses: actions/upload-artifact@v3
        with:
          name: xray-scan
          path: reports/xray-scan.json
```

You can download the artifact from the **Actions run** page.  
Local path in the runner: `reports/xray-scan.json`.

✅ After this, you’ll have JSON scan results available as workflow artifacts.

---

## 🧭 How to Run the Project

### Option A — Local Maven
```bash
mvn package
java -jar target/spring-petclinic-*.jar
# http://localhost:8080
```

### Option B — Local Docker (build locally)
```bash
mvn package
docker build -t spring-petclinic .
docker run --rm -p 8080:8080 spring-petclinic
# http://localhost:8080
```

### Option C — Pull from JFrog (your instance)
```bash
docker pull trialgh4oxk.jfrog.io/docker-local/spring-petclinic:latest
docker run --rm -p 8080:8080 trialgh4oxk.jfrog.io/docker-local/spring-petclinic:latest
# http://localhost:8080
```

---

## 🧰 Tips & Troubleshooting

- **Port in use**: change mapping, e.g., `-p 8081:8080`.
- **Docker login 401**: re-check `JFROG_USERNAME / JFROG_PASSWORD (token)` and repo permissions.
- **Slow first Docker build**: normal (downloads Maven deps). Subsequent builds are cached.
- **Keep images small**: consider multi-stage builds and a JRE base image.
- **.dockerignore** (recommended):
  ```
  target/
  .git/
  .github/
  .idea/
  *.iml
  ```

---

## 📦 Deliverables Checklist

- ✅ GitHub repo link (your fork)
- ✅ `.github/workflows/build.yml`
- ✅ `Dockerfile`
- ✅ `README.md`
- ✅ `reports/xray-scan.json` (as workflow artifact)
- ✅ JFrog Docker run command (documented above)

---

## 📝 Notes

- This project uses **Java 21** and assumes **Maven** wraps build/test.
- The GitHub Actions workflow runs on **push** and **pull_request** to `main`.
- Keep **secrets in GitHub Actions**, never commit credentials.

---


