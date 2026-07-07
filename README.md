# BankApp

This repository contains a simple Spring Boot-based banking web application. The app demonstrates user registration, login, deposit, withdrawal, transfer, and transaction history functionality. It is intended as a demo / learning project for Java web development and DevOps workflows (containerization, Kubernetes deployment, CI/CD integration points).

## Features

- User registration and login (Spring Security)
- Account balance management (deposit, withdraw)
- Transfer money between users
- Transaction history per account
- Thymeleaf-based UI pages
- JPA persistence using MySQL

## Tech Stack

- Java 17
- Spring Boot (Web, Data JPA, Security, Thymeleaf)
- Spring Data JPA (Hibernate)
- MySQL (runtime connector `mysql-connector-java`)
- Maven (wrapper included: `mvnw` / `mvnw.cmd`)
- Docker (via `Dockerfile`)
- Kubernetes manifest (`ds.yml`) for Deployment and Service
- JaCoCo for test coverage (configured in `pom.xml`)

## Repository Layout

- `.github/workflows/cicd.yml` — GitHub Actions pipeline for compile, security scan, tests, Docker build/push, and `ds.yml` update
- `.gitignore` — ignored files
- `.mvn/` — Maven wrapper support files
- `Dockerfile` — container image build definition
- `ds.yml` — Kubernetes manifest for MySQL and the bankapp Deployment/Services
- `mvnw` / `mvnw.cmd` — Maven wrapper scripts
- `pom.xml` — Maven build configuration, dependencies, JaCoCo, and distributionManagement
- `README.md` — project documentation
- `Setup-RBAC.md` — Kubernetes RBAC sample for service account and role binding
- `sonar-project.properties` — SonarQube project settings
- `src/` — application source code and resources

### Source code structure

- `src/main/java/com/example/bankapp/`
  - `BankappApplication.java` — Spring Boot entry point
  - `config/SecurityConfig.java` — Spring Security setup
  - `controller/BankController.java` — web controller endpoints and request handling
  - `service/AccountService.java` — business logic, user details service, deposit/withdraw/transfer
  - `model/Account.java`, `Transaction.java` — JPA entity mappings and security model
  - `repository/AccountRepository.java`, `TransactionRepository.java` — Spring Data JPA interfaces
- `src/main/resources/templates/` — Thymeleaf HTML templates (`dashboard.html`, `login.html`, `register.html`, `transactions.html`)
- `src/main/resources/application.properties` — application configuration (datasource, JPA)
- `src/main/resources/static/mysql/SQLScript.txt` — database creation SQL script

## How the application works (end-to-end)

1. User visits the app in a browser (`/login` or `/register`). The UI uses Thymeleaf templates.
2. New users register via `/register`. Passwords are hashed using BCrypt by `AccountService`.
3. Users log in via Spring Security; authenticated sessions are maintained by Spring Security.
4. After login, users are redirected to `/dashboard` where the current balance and account details are shown.
5. Users can submit forms to deposit or withdraw money. The controller calls `AccountService` to update the `Account` balance and create `Transaction` entries.
6. Transfers between users are implemented via `AccountService.transferAmount()`, which atomically adjusts balances and creates transaction records for both accounts.
7. All persistence is handled by Spring Data JPA into a MySQL database. Entities are mapped with JPA annotations.

## Local development: prerequisites

- Java 17 JDK
- Maven (or use included `mvnw`/`mvnw.cmd`)
- MySQL (or run MySQL in Docker/Kubernetes)
- Docker (optional, for container tests)

## Build and run locally

1. Create the MySQL database used by the app (default name: `bankappdb`). You can run the SQL from `src/main/resources/static/mysql/SQLScript.txt` or let the app create schema using `spring.jpa.hibernate.ddl-auto=update`.

2. Build the project:

```bash
./mvnw clean package
```

3. Run the jar:

```bash
java -jar target/*.jar
```

4. Open `http://localhost:8080/login` in a browser.

## Docker

The `Dockerfile` expects the packaged jar in `target/*.jar` and copies it into the image.

Build and run locally:

```bash
# Build jar
./mvnw clean package

# Build Docker image (run from repo root)
docker build -t bankapp:local .

# Run container (connect to a local MySQL instance or provide env vars)
docker run --rm -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:mysql://host.docker.internal:3306/bankappdb \
  -e SPRING_DATASOURCE_USERNAME=root \
  -e SPRING_DATASOURCE_PASSWORD=Test@123 \
  bankapp:local
```

Note: On Linux, `host.docker.internal` may not be available. Use the Docker network or run MySQL in Docker and link networks.

## Kubernetes (`ds.yml`)

- `ds.yml` is a multi-document Kubernetes manifest that:
  - Deploys a MySQL pod (image `mysql:5.7`) with `MYSQL_ROOT_PASSWORD=Test@123` and `MYSQL_DATABASE=bankappdb`.
  - Exposes MySQL through a `Service` named `mysql-service`.
  - Deploys the `bankapp` Deployment using image `adijaiswal/bankapp:latest` and sets environment variables so the app connects to `mysql-service`.
  - Exposes the `bankapp` via a `Service` of type `LoadBalancer` mapping port `80` to container `8080`.

Apply manifest to a cluster:

```bash
kubectl apply -f ds.yml
```

Important: `ds.yml` currently contains plaintext DB password and does not use PVCs for MySQL persistence. For production, use `Secrets` and `PersistentVolumeClaim`.

## CI/CD and DevOps notes

- This repo includes a GitHub Actions workflow in `.github/workflows/cicd.yml`.
- The pipeline triggers on pushes to the `main` branch and includes four linked jobs:
  1. `compile` — checks out the repository, installs JDK 17, and runs `mvn compile`.
  2. `security-check` — performs a Trivy filesystem scan and Gitleaks secret detection.
  3. `test` — runs unit tests with `mvn test`.
  4. `build-package-and-push` — builds the project, logs into Docker Hub, builds and pushes a Docker image, tags it as `latest`, and updates `ds.yml` with the new image tag.
- `build-package-and-push` uses GitHub secrets `DOCKER_USER` and `DOCKER_PASSWORD` for Docker Hub login.
- The workflow also uses `permissions: contents: write` so it can commit and push changes to `ds.yml` after updating the image tag.
- `pom.xml` additionally includes JaCoCo configuration and `distributionManagement` entries for Nexus-style artifact publishing.
- The current CI pipeline is functional and can be extended with SonarQube scanning, artifact publishing, and deployment automation.

### How the CICD workflow runs

1. Push code to `main`.
2. GitHub Actions starts the `compile` job:
   - checkout source
   - set up JDK 17
   - run `mvn compile`
3. If compile passes, `security-check` runs:
   - Trivy filesystem scan
   - Gitleaks secret scan
4. If security checks pass, `test` runs:
   - checkout source
   - set up JDK 17
   - run `mvn test`
5. If tests pass, `build-package-and-push` runs:
   - build jar with `mvn package`
   - login to Docker Hub with GitHub secrets
   - build image `DOCKER_USER/bankapp:${{ github.run_id }}`
   - push the image and `latest` tag
   - update `ds.yml` to use the new image tag and commit it back to the repo

## Security and production-readiness checklist

- Do NOT keep database credentials in plaintext inside manifests or `application.properties`. Use Kubernetes `Secret` or CI/CD secrets.
- Add `readinessProbe` and `livenessProbe` to the app `Deployment`.
- Add `PersistentVolumeClaim` for MySQL to persist data.
- Replace `spring.jpa.hibernate.ddl-auto=update` with managed schema migrations (Flyway or Liquibase) for production.
- Configure HTTPS / ingress with TLS for secure traffic.

## Next steps I can help with

- Add a GitHub Actions workflow to run build, tests, JaCoCo, Sonar scan, and Docker build/push.
- Convert `ds.yml` to use `Secrets` + `PVC` + readiness/liveness probes or create a Helm chart.
- Add a sample `.github/workflows/ci.yml` and show how to store registry/Nexus credentials as GitHub Secrets.

---

If you want, I can now create a CI workflow file (`.github/workflows/ci.yml`) that runs tests, collects JaCoCo, builds a Docker image, and pushes it to Docker Hub. Tell me the Docker repository/name and whether you want Sonar or Nexus publishing included.
