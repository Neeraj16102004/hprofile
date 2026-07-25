# Automated Java/Maven CI/CD Pipeline to AWS ECR, ECS & RDS

An end-to-end continuous integration and deployment (CI/CD) pipeline built with **GitHub Actions** for the `Hprofile` Java application. This pipeline automates code quality analysis with **SonarQube / SonarCloud**, Maven testing and Checkstyle analysis, dynamic database configuration, containerization with **Docker**, and automated publishing to **Amazon ECR** for deployment to **Amazon ECS** and **Amazon RDS**.

---

## 📐 Pipeline Architecture Overview
```text
+------------------+
|  VS Code / Git   |
+--------+---------+
|  git push / manual trigger
v
+-----------------------------------------------------------------------------------+
| GitHub Repository                                                                 |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  | GitHub Actions Workflow (Hprofile Actions)                                |  |
|  |                                                                             |  |
|  |  +-----------------------------------------------------------------------+  |  |
|  |  | Job 1: Testing                                                        |  |  |
|  |  |  1. Code checkout                                                     |  |  |
|  |  |  2. Maven test                                                        |  |  |
|  |  |  3. Checkstyle (mvn checkstyle:checkstyle)                           |  |  |
|  |  |  4. Set Java 21 (Temurin JDK)                                         |  |  |
|  |  |  5. SonarQube Scan (Sonar Maven Plugin) <=> SonarQube / SonarCloud|  |  |
|  |  +-----------------------------------+-----------------------------------+  |  |
|  |                                      |                                      |  |
|  |                                      v                                      |  |
|  |  +-----------------------------------------------------------------------+  |  |
|  |  | Job 2: BUILD_AND_PUBLISH                                              |  |  |
|  |  |  1. Code checkout                                                     |  |  |
|  |  |  2. Inject RDS Credentials into application.properties via sed      |  |  |
|  |  |  3. Build & Upload Docker image (actapp) to Amazon ECR              |  |  |
|  |  +-------------------+--------------------------------+------------------+  |  |
|  +----------------------|--------------------------------|---------------------+  |
+-------------------------|--------------------------------|------------------------+
|                                |
| Push Image (actapp)          | Deploy Update
v                                v
+------------------------------------------------------------------------+
| AWS Cloud (us-east-1)                                                |
|                                                                        |
|     +---------------+                  +--------------------------+    |
|     |  Amazon ECR   |                  |        Amazon ECS        |    |
|     |   (actapp)  |                  |  +--------------------+  |    |
|     +---------------+                  |  |   ECS Container    |<> ELB
|                                        |  +---------+----------+  |    |
|                                        +------------|-------------+    |
|                                                     | DB Connection    |
|                                                     v                  |
|                                        +--------------------------+    |
|                                        |        Amazon RDS        |    |
|                                        +--------------------------+    |
+------------------------------------------------------------------------+
```
---

## Flow Summary

1. **Developer Workflow:** Source code changes are pushed to GitHub or triggered manually (`workflow_dispatch`).
2. **Testing Job (`Testing`):**
   * **Code Checkout:** Pulls code using `actions/checkout@v4`.
   * **Maven Unit Testing:** Runs `mvn test`.
   * **Checkstyle Validation:** Executes `mvn checkstyle:checkstyle`.
   * **Java Setup:** Configures Temurin Java 21 distribution.
   * **SonarQube Scan:** Runs `sonar-maven-plugin` analyzing unit test results, JaCoCo execution reports, and Checkstyle metrics.
3. **Build & Publish Job (`BUILD_AND_PUBLISH`):**
   * **Inject Environment Configurations:** Replaces target database host, username, and password in `src/main/resources/application.properties` dynamically using `sed`.
   * **Docker Build & ECR Push:** Uses `appleboy/docker-ecr-action` to build the Docker image and push tags (`latest` and `${{ github.run_number }}`) to the ECR repository (`actapp`).
4. **AWS Infrastructure:**
   * **Amazon ECR:** Stores `actapp` container images tagged per workflow run.
   * **Amazon ECS & ELB:** Pulls the image and serves traffic via an Elastic Load Balancer.
   * **Amazon RDS:** Manages backend persistent storage initialized via injected connection properties.

---

## 🛠️ Tech Stack & Tools

* **IDE / Source Control:** Visual Studio Code, Git, GitHub
* **CI/CD Automation:** GitHub Actions (`workflow_dispatch`)
* **Build & Quality Assurance:** Java 21, Apache Maven, Checkstyle, SonarQube / SonarCloud
* **Containerization:** Docker (`./Dockerfile`)
* **AWS Services:**
  * **Amazon ECR:** Container image registry (Repository: `actapp`)
  * **Amazon ECS:** Application orchestration
  * **Amazon RDS:** Relational Database Instance
  * **ELB:** Elastic Load Balancer

---

## 🔑 Configured GitHub Secrets

Configure the following secrets in your repository (**Settings > Secrets and variables > Actions**):

| Secret Name | Description |
| :--- | :--- |
| `AWS_ACCESS_KEY_ID` | AWS IAM Access Key ID with ECR push permissions |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM Secret Access Key |
| `REGISTRY` | AWS ECR Registry Endpoint (e.g., `<account-id>.dkr.ecr.us-east-1.amazonaws.com`) |
| `RDS_USER` | Database username injected into `application.properties` |
| `RDS_PASS` | Database password injected into `application.properties` |
| `RDS_ENDPOINT` | RDS instance host replacing `db01` in properties |
| `SONAR_URL` | Base URL for the SonarQube / SonarCloud instance |
| `SONAR_TOKEN` | Authentication token for SonarQube |
| `SONAR_ORGANIZATION` | SonarCloud organization key |
| `SONAR_PROJECT_KEY` | Unique project key defined in SonarQube |

---

## ⚙️ CI/CD Workflow (`.github/workflows/hprofile.yml`)

```yaml
name: Hprofile Actions
on: workflow_dispatch
env:
  AWS_REGION: us-east-1

jobs:
  Testing:
    runs-on: ubuntu-latest
    steps:
      - name: Code checkout
        uses: actions/checkout@v4

      - name: Maven test
        run: mvn test

      - name: Checkstyle
        run: mvn checkstyle:checkstyle

      - name: Set Java 21
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'

      - name: SonarQube Scan
        run: |
          mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
            -Dsonar.host.url="${{ secrets.SONAR_URL }}" \
            -Dsonar.token="${{ secrets.SONAR_TOKEN }}" \
            -Dsonar.organization="${{ secrets.SONAR_ORGANIZATION }}" \
            -Dsonar.projectKey="${{ secrets.SONAR_PROJECT_KEY }}" \
            -Dsonar.junit.reportsPath=target/surefire-reports/ \
            -Dsonar.jacoco.reportsPath=target/jacoco.exec \
            -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml \
            -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/

  BUILD_AND_PUBLISH:
    needs: Testing
    runs-on: ubuntu-latest
    steps:
      - name: Code checkout
        uses: actions/checkout@v4

      - name: Update application.properties file
        run: |
          sed -i "s/^jdbc.username.*$/jdbc.username\=${{ secrets.RDS_USER }}/" src/main/resources/application.properties
          sed -i "s/^jdbc.password.*$/jdbc.password\=${{ secrets.RDS_PASS }}/" src/main/resources/application.properties
          sed -i "s/db01/${{ secrets.RDS_ENDPOINT }}/" src/main/resources/application.properties

      - name: Build & Upload image to ECR
        uses: appleboy/docker-ecr-action@master
        with:
          access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          registry: ${{ secrets.REGISTRY }}
          repo: actapp
          region: ${{ env.AWS_REGION }}
          tags: latest,${{ github.run_number }}
          daemon_off: false
          dockerfile: ./Dockerfile
          context: ./
```
---

## 🔒 Security Best Practices Implemented

Dynamic Secret Injection: Database hostnames and passwords are never hardcoded into source code; they are dynamically replaced at build time via GitHub Secrets.

Network & Database Security: RDS access credentials stay isolated within GitHub Actions execution contexts.

Code Quality Gates: Automated Maven testing, Checkstyle enforcement, and SonarQube static code analysis execute before container generation.

---
