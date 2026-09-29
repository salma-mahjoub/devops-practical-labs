# Student Management - DevOps Practical Labs

> A Spring Boot student-management REST API used to practise a complete DevOps delivery path: build, containerise, deploy to Kubernetes, and monitor with Prometheus and Grafana.

## Context

This repository is practical coursework for a university DevOps module. It uses a small student-management application as the workload for hands-on exercises in CI/CD, containers, Kubernetes, and observability.

The repository is organised around the artefacts produced for each practical lab rather than separate numbered lab directories:

| Practical lab | Focus | Documented artefacts |
| --- | --- | --- |
| Application baseline | REST API and persistence | `src/`, `pom.xml` |
| Containerisation | Package the application as a runnable image | `Dockerfile` |
| CI/CD | Build, publish, and deploy through Jenkins | `Jenkinsfile` |
| Kubernetes deployment | Run the application and MySQL in the `devops` namespace | `k8s/` |
| Monitoring | Scrape application and cluster metrics, then visualise them | `k8s/monitoring/` |


## Stack

- Java 17 and Spring Boot 3.5.5
- Spring Web and Spring Data JPA
- MySQL 8.0
- Maven and JUnit 5 / Spring Boot Test
- Lombok
- Spring Boot Actuator, Micrometer, and Prometheus registry
- Docker
- Jenkins Pipeline
- Kubernetes (Deployments, Services, ConfigMaps, Secrets, PV/PVC, RBAC)
- Prometheus and Grafana
- Springdoc OpenAPI UI dependency

## Features

- CRUD endpoints for students, departments, and enrolments.
- JPA domain model for students, departments, courses, and enrolments.
- Enrolment statuses: `ACTIVE`, `COMPLETED`, `DROPPED`, `FAILED`, and `WITHDRAWN`.
- Configurable MySQL connection using `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, and `SPRING_DATASOURCE_PASSWORD`.
- Actuator endpoints, including Prometheus metrics at `/student/actuator/prometheus`.
- Docker image definition exposing port `8089`.
- Kubernetes manifests for a two-replica Spring application and a persistent MySQL deployment.
- Prometheus configured to scrape the Spring Boot metrics endpoint; Grafana provisioned with Prometheus as its default datasource.

The project includes a `Course` entity and `CourseRepository`, but no course controller or service is currently implemented.

## Architecture

```text
REST clients
    |
    v
Spring Boot API (:8089, context path /student)
    |
    +-- Controllers -> Services -> Spring Data JPA repositories
    |                                  |
    |                                  v
    |                               MySQL (:3306)
    |
    +-- Actuator / Prometheus metrics
                 |
                 v
          Prometheus (:9090) -> Grafana (:3000)
```

Within Kubernetes, the application is exposed by the `spring-service` NodePort service on `30080`. MySQL is internal through `mysql-service`. Prometheus and Grafana are exposed through NodePorts `30090` and `30030` respectively.

## API Surface

All API paths are prefixed with `/student`.

| Resource | Base path | Operations |
| --- | --- | --- |
| Students | `/students` | `GET /getAllStudents`, `GET /getStudent/{id}`, `POST /createStudent`, `PUT /updateStudent`, `DELETE /deleteStudent/{id}` |
| Departments | `/Depatment` | `GET /getAllDepartment`, `GET /getDepartment/{id}`, `POST /createDepartment`, `PUT /updateDepartment`, `DELETE /deleteDepartment/{id}` |
| Enrolments | `/Enrollment` | `GET /getAllEnrollment`, `GET /getEnrollment/{id}`, `POST /createEnrollment`, `PUT /updateEnrollment`, `DELETE /deleteEnrollment/{id}` |

Note: `/Depatment` is intentionally documented with the spelling used by the current controller.

## Prerequisites

- JDK 17
- Maven 3.9+ (or a compatible Maven installation)
- MySQL for local execution
- Docker for image builds
- A Kubernetes cluster and `kubectl` for deployment
- Jenkins, Docker Hub credentials, and cluster access for the pipeline

## Install and Run Locally

1. Create a local MySQL database. The default connection is `jdbc:mysql://localhost:3306/studentdb?createDatabaseIfNotExist=true`, with user `root` and an empty password.
2. Override the defaults when needed:

```bash
export SPRING_DATASOURCE_URL='jdbc:mysql://localhost:3306/studentdb?createDatabaseIfNotExist=true'
export SPRING_DATASOURCE_USERNAME='root'
export SPRING_DATASOURCE_PASSWORD='your-password'
```

3. Build and run the application:

```bash
mvn clean package
mvn spring-boot:run
```

The application listens on `http://localhost:8089/student`.

Useful checks:

```bash
curl http://localhost:8089/student/actuator/health
curl http://localhost:8089/student/actuator/prometheus
```

Run the existing test suite with:

```bash
mvn test
```

## Build and Run the Container

The Dockerfile expects the JAR to have already been produced in `target/`.

```bash
mvn clean package
docker build -t student-management:local .
docker run --rm -p 8089:8089 \
  -e SPRING_DATASOURCE_URL='jdbc:mysql://host.docker.internal:3306/studentdb?createDatabaseIfNotExist=true' \
  -e SPRING_DATASOURCE_USERNAME='root' \
  -e SPRING_DATASOURCE_PASSWORD='your-password' \
  student-management:local
```

## Kubernetes Deployment

The manifests target a namespace named `devops`; it is referenced but not created by the repository. Create or select a suitable namespace before applying them.

```bash
kubectl create namespace devops
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/mysql-deployment.yaml
kubectl apply -f k8s/spring-deployment.yaml
```

Deploy monitoring separately:

```bash
kubectl apply -f k8s/monitoring/prometheus-configmap.yaml
kubectl apply -f k8s/monitoring/prometheus-deployment.yaml
kubectl apply -f k8s/monitoring/grafana-deployment.yaml
```

Inspect the rollout:

```bash
kubectl get pods,services -n devops
kubectl rollout status deployment/spring-app -n devops
```

The MySQL manifest uses a `hostPath` persistent volume at `/data/mysql` and the `standard` storage class. These settings are cluster-specific and may need adapting outside a local Kubernetes environment.

## CI/CD Pipeline

`Jenkinsfile` defines the following stages:

1. Checkout the `main` branch from the repository URL configured in the pipeline.
2. Run `mvn clean package -DskipTests`.
3. Build and tag a Docker image with the Jenkins build number and `latest`.
4. Push both tags to Docker Hub using the Jenkins credential ID `dockerhub-credentials`.
5. Apply the MySQL and Spring Kubernetes manifests, then restart `spring-app` in the `devops` namespace.

Monitoring manifests are present in the repository but are not applied by the current Jenkins pipeline.

## Project Structure

```text
.
├── src/
│   ├── main/
│   │   ├── java/tn/esprit/studentmanagement/
│   │   │   ├── controllers/       # REST endpoints
│   │   │   ├── entities/          # JPA domain model
│   │   │   ├── repositories/      # Spring Data repositories
│   │   │   └── services/          # Application services
│   │   └── resources/
│   │       └── application.properties
│   └── test/                      # Spring Boot context-load test
├── k8s/
│   ├── monitoring/                # Prometheus and Grafana manifests
│   ├── mysql-deployment.yaml
│   ├── secrets.yaml
│   └── spring-deployment.yaml
├── Dockerfile
├── Jenkinsfile
└── pom.xml
```

