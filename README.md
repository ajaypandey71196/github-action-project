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

## Project Structure

- `pom.xml` — Maven build configuration (dependencies, JaCoCo, distributionManagement)
- `Dockerfile` — builds a Docker image that runs the packaged jar
- `ds.yml` — Kubernetes manifest (multi-document) that deploys MySQL and the `bankapp` Deployment + Services
- `src/main/java/com/example/bankapp/` — application sources
  - `BankappApplication.java` — Spring Boot entry point
  - `config/SecurityConfig.java` — Spring Security setup
  - `controller/BankController.java` — web controllers and endpoints
  - `service/AccountService.java` — business logic and `UserDetailsService` implementation
  - `model/Account.java`, `Transaction.java` — JPA entities
  - `repository/AccountRepository.java`, `TransactionRepository.java` — Spring Data repositories
- `src/main/resources/templates/` — Thymeleaf HTML templates (`dashboard.html`, `login.html`, `register.html`, `transactions.html`)
- `src/main/resources/application.properties` — default app configuration (datasource, JPA)
- `src/main/resources/static/mysql/SQLScript.txt` — SQL to create the database

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

- The repo contains references for common CI/CD and quality tools but no workflow files under `.github/workflows/` yet.
- `pom.xml` contains JaCoCo plugin configuration and `distributionManagement` entries for publishing to a Maven repository (Nexus/Artifactory).
- Typical CI pipeline to add:
  1. `mvn -B -DskipTests=false clean verify` — run compile and tests
  2. Run JaCoCo coverage report and send to SonarQube
  3. Build Docker image and push to registry (Docker Hub or internal registry)
  4. Deploy to Kubernetes (e.g. via `kubectl` or Helm) or trigger a CD tool (ArgoCD/Flux)

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
