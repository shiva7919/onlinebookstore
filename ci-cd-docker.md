# Complete CI/CD Pipeline — Jenkins, SonarQube, Nexus, Docker, DockerHub, Tomcat (Detailed Step-by-Step)

> Goal: Provide a neat, clear, and deeply explanatory step-by-step guide to build a full CI/CD pipeline for a Java web app (WAR) using Jenkins, SonarQube, Nexus, Docker, DockerHub and Tomcat on an AWS EC2 Ubuntu instance.

---

## Table of contents

1. Overview & architecture diagram
2. Prerequisites & assumptions
3. EC2 instance creation & network/security
4. System preparation (user, packages, Docker)
5. SonarQube (run, secure, usage)
6. Nexus Repository (run, initial setup, repos)
7. Building a Jenkins image with Docker inside
8. Running Jenkins container and initial setup
9. Jenkins global tool & plugin configuration (detailed)
10. Credentials and secrets management (best practices)
11. Maven `pom.xml` and `settings.xml` for Nexus deployment
12. The Jenkinsfile explained — stage-by-stage (with sample)
13. Building & pushing Docker images to DockerHub
14. Tomcat Dockerfile and deployment steps
15. Putting it all together — pipeline flow and triggers
16. Troubleshooting (common issues & fixes)
17. Security hardening & production considerations
18. Backup, scaling and monitoring suggestions
19. Appendix: useful commands & references

---

## 1. Overview & architecture diagram

**Short description:** Source code in GitHub triggers a Jenkins pipeline. Jenkins performs build & unit tests, runs SonarQube analysis, deploys artifacts to Nexus, builds a Docker image, pushes it to DockerHub and finally deploys to a Tomcat container on the target host.

(Include a simple ASCII diagram or insert your own image when publishing.)

```
GitHub -> Jenkins -> [Maven build -> Unit tests]
                          -> SonarQube (static analysis)
                          -> Nexus (artifact deploy)
                          -> Docker build -> DockerHub
                          -> Tomcat (deploy WAR)
```

---

## 2. Prerequisites & assumptions

* You have an AWS account and permission to create EC2 instances and security groups.
* A basic understanding of Linux shell, Docker, and Maven.
* Java project producing a WAR (packaging: war).
* GitHub repository containing the project and `Jenkinsfile` in repo root.
* This guide uses Ubuntu 22.04 on an `t2.large` instance as an example.

---

## 3. EC2 instance creation & network/security

**Recommended instance:** `t2.large` (2 vCPU, 8 GiB RAM) — adjust memory if SonarQube or other services need more.

**Storage:** 16 GB minimum — increase for logs, artifacts, docker images (50+ GB recommended for CI servers).

**Security Group (open ports):**

* 22 (SSH) — source: your IP
* 8080 (Jenkins) — restrict to admin IPs if public
* 8081 (Nexus)
* 9000 (SonarQube)
* 8082 (Tomcat app)
* Optionally 2375 (Docker remote) — avoid exposing this publicly; prefer socket file.

> TIP: Limit access by IP (not 0.0.0.0/0) and use a bastion host if needed.

---

## 4. System preparation (user, packages, Docker)

Perform these steps after launching the EC2 instance and SSHing in.

```bash
# update packages
sudo apt update -y && sudo apt upgrade -y

# create a dedicated admin user (optional)
sudo adduser devops
sudo usermod -aG sudo devops

# install Docker
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu  # or your user
# Log out & back in for group changes to take effect

# install docker-compose (optional)
sudo apt install -y docker-compose
```

**Verify Docker:**

```bash
docker version
docker run --rm hello-world
```

---

## 5. SonarQube (run, secure, usage)

**Why SonarQube?** Static code analysis, code smells, vulnerabilities, maintainability.

**Minimum memory:** SonarQube likes RAM — 2 GB+; production 4+ GB recommended.

**Run SonarQube using Docker:**

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts
```

**Access:** `http://<EC2_PUBLIC_IP>:9000` (default admin/admin). Change password, generate a token for Jenkins and save it in Jenkins credentials as `sonar-token`.

**Important:** If Sonar fails due to memory, increase Docker host memory or run with custom `-e` settings or use an official compose setup.

---

## 6. Nexus Repository (run, initial setup, repos)

**Purpose:** Store maven artifacts (releases/snapshots) and act as a proxy for public repos.

**Run Nexus:**

```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
```

**Retrieve initial admin password:**

```bash
docker exec -it nexus cat /nexus-data/admin.password
```

**Setup steps in UI:**

1. Login to `http://<EC2_PUBLIC_IP>:8081` with `admin` + password.
2. Change password.
3. Create `maven-releases` and `maven-snapshots` hosted repositories (if not present).
4. Create a user (e.g., `ci-user`) with deploy privileges or use admin for quick tests.

**Maven distributionManagement** in `pom.xml` later should point to the releases repository URL.

---

## 7. Building a Jenkins image with Docker inside

**Why:** Jenkins must access Docker to build/push images. Mounting the host Docker socket or installing Docker inside the Jenkins container are two approaches. Mounting the socket is simpler; installing Docker inside the container may be preferred for portability.

**Sample Dockerfile (jenkins-docker):**

```dockerfile
FROM jenkins/jenkins:lts
USER root
RUN apt-get update && apt-get install -y docker.io
RUN groupadd -g 999 docker || true
RUN usermod -aG docker jenkins
USER jenkins
```

**Build image:**

```bash
docker build -t jenkins-docker:latest .
```

**Run container (recommended with volume + socket mount):**

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins-docker:latest
```

**Why mount socket?** Allows Jenkins to call Docker on the host; no need for Docker-in-Docker. Be mindful: this gives the container root-equivalent access to Docker host.

---

## 8. Running Jenkins container and initial setup

**Get initial admin password:**

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Initial UI steps:**

1. Open `http://<EC2_PUBLIC_IP>:8080`.
2. Enter initial password.
3. Install suggested plugins (or install manually: Git, Pipeline, Docker Pipeline, SonarQube Scanner, Maven Integration, Credentials Binding, GitHub plugin, etc.).
4. Create first admin user.

**Create a node/agent (optional):** For heavy builds, use additional Docker nodes or separate EC2 instances as agents.

---

## 9. Jenkins global tool & plugin configuration (detailed)

**Global Tool Configuration:**

* **JDK:** Add `JDK17` pointing to the installed JDK or configure automatic install.
* **Maven:** Add `Maven-3` and enable automatic install, or point to a preinstalled Maven.

**Manage Plugins (important list):**

* Pipeline
* Git
* Maven Integration
* Docker Pipeline
* SonarQube Scanner
* Credentials Binding
* JUnit
* GitHub Integration or GitHub Branch Source (for multibranch)
* Nexus Artifact Uploader (optional)

**SonarQube server config:** Manage Jenkins → Configure System → SonarQube servers → Add server. Provide name and server URL; leave token blank here if using pipeline credentials.

---

## 10. Credentials and secrets management (best practices)

**Credential IDs used in examples:**

* `sonar-token` — Secret text
* `nexus` — Username/password
* `dockerhub-user` — Username/password

**How to add:** Jenkins → Credentials → System → Global credentials → Add credentials.

**Best practices:**

* Use least privilege accounts.
* Rotate tokens regularly.
* Do NOT store secrets in pipeline code; use credentials plugin and `withCredentials` blocks.
* Use Jenkins’ credential store or an external vault (HashiCorp Vault, AWS Secrets Manager) for production.

---

## 11. Maven `pom.xml` and `settings.xml` for Nexus deployment

**`pom.xml` snippet (distributionManagement)**

```xml
<distributionManagement>
  <repository>
    <id>nexus</id>
    <url>http://<EC2_PUBLIC_IP>:8081/repository/maven-releases/</url>
  </repository>
</distributionManagement>
```

**`settings.xml` to provide credentials (generated dynamically by pipeline):**

```xml
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
```

**Pipeline will create `settings.xml` at runtime** passing credentials via environment variables — this prevents committing credentials to git.

---

## 12. The Jenkinsfile explained — stage-by-stage (with sample)

Below is a clean sample Jenkinsfile and a precise explanation for each stage. The full sample `Jenkinsfile` is included in the canvas document (and in your repository). The pipeline uses declarative syntax and the `withCredentials` steps.

**Conceptual stages:**

* Checkout: Pull code from SCM
* Build & Test: `mvn clean package` + store JUnit results
* SonarQube Analysis: Run `mvn sonar:sonar` with token
* Upload to Nexus: `mvn deploy` with runtime `settings.xml`
* Build Docker Image: `docker.build()` using `Dockerfile` from repo
* Push to DockerHub: Login & `docker push`
* (Optional) Deploy to target: Pull image & run Tomcat container (or deploy WAR via other means)

**Explanation tips:**

* Use `post { always {}}` blocks to always collect artifacts or clean workspace.
* Use `allowEmptyResults` with `junit` when tests may not exist during initial commits.
* Use `VERSION = "${env.BUILD_NUMBER}"` or calculate semantic versions from `git describe` or tags for production.

---

## 13. Building & pushing Docker images to DockerHub

**Dockerfile for app (example):**

```dockerfile
FROM tomcat:9-jdk17
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/onlinebookstore.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

**Pipeline snippet (login & push):**

* Use `withCredentials([usernamePassword(credentialsId: 'dockerhub-user', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')])` and pipe password to `docker login --password-stdin`.

**Tagging strategy:**

* Tag by build number: `image:version` and also tag `latest`.
* For production, use semantic versions and signed images if required.

---

## 14. Tomcat Dockerfile and deployment steps

Two deployment options:

1. Build a Tomcat image containing the WAR and run it.
2. Keep WAR in Nexus and use a deployment script on the target host to pull artifact and place into Tomcat.

**Option 1 (containerized Tomcat):** Built earlier in the Dockerfile section. Run with:

```bash
docker run -d -p 8082:8080 --name bookstore kishangollamudi/onlinebookstore:latest
```

**Option 2 (artifact deploy):** On target server, `curl` or `wget` the WAR from Nexus (if repository allows) and copy to Tomcat `webapps/ROOT.war`.

---

## 15. Putting it all together — pipeline flow and triggers

**Triggering strategies:**

* Webhooks from GitHub to Jenkins (recommended for fast CI)
* Poll SCM (not recommended if webhooks available)
* Scheduled builds for nightly jobs

**Multibranch pipelines:** Use GitHub Branch Source plugin and a `Jenkinsfile` per branch.

**Promotion:** For production workflows, add a manual `Approve` stage or use separate CI/CD pipelines for dev/staging/prod.

---

## 16. Troubleshooting (common issues & fixes)

**`Tool type "maven" does not have an install of "Maven-3" configured`**

* Ensure Jenkins Global Tool Configuration has a Maven with the exact name used in the `Jenkinsfile` (`Maven-3`).

**Docker permission errors**

* If `docker` commands fail in Jenkins, ensure the jenkins user is in the docker group or `/var/run/docker.sock` is mounted and has appropriate permissions.

**SonarQube memory or startup failure**

* Allocate more memory to host or use `sonar` docker-compose recommended settings.

**Nexus 401/403 on deploy**

* Verify `settings.xml` credentials, repository URL and that the repository accepts deploy for the repo id used in `pom.xml`.

**Maven cannot download plugins**

* Ensure Nexus proxy is configured or Maven settings allow direct access to central (or add proxy settings).

---

## 17. Security hardening & production considerations

* Do not expose Docker socket or management ports publicly.
* Use SSL/TLS on Jenkins, Nexus, SonarQube (reverse proxy like Nginx with certs).
* Use IAM roles, VPC, private subnets for internal services.
* Use secrets manager (Vault, AWS Secrets Manager) for credentials.
* Audit logs and enable backups for Jenkins home and Nexus data (`/nexus-data`).

---

## 18. Backup, scaling and monitoring suggestions

* Backup Jenkins `/var/jenkins_home` regularly (jobs, plugins, credentials backup strategy).
* Backup Nexus data volume `/nexus-data`.
* Use Prometheus + Grafana for metrics (Jenkins exporter, Docker metrics).
* For scale, convert services to separate EC2/ECS services or Kubernetes.

---

## 19. Appendix: useful commands & references

**Docker:**

```bash
docker ps
docker logs <container>
docker exec -it <container> bash
docker volume ls
docker image prune -a
```

**Jenkins Troubleshooting:**

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
# tail logs
docker logs -f jenkins
```

**Maven deploy debug:**

```bash
mvn -X deploy -s settings.xml
```

---

## Next steps I can do for you (pick any)

* Convert this document into a polished README with images and commands formatted for README usage.
* Create a ready-to-use `docker-compose.yml` to bring up Jenkins, Sonar, Nexus locally for demo.
* Produce a production-ready Jenkinsfile with promotion and approvals.
* Create Terraform scripts to provision the EC2 + security groups.

---

*End of document.*

---
# 🎉 **Final Outcome**

You now have a **complete CI/CD pipeline**:

✔ GitHub → Jenkins → SonarQube → Nexus → Docker → DockerHub → Tomcat

✔ Fully automated WAR build
✔ Static code analysis
✔ Artifact upload
✔ Docker image build
✔ Push to DockerHub
✔ Deployment in Tomcat container

---
# Screenshots
<img width="1920" height="1080" alt="32" src="https://github.com/user-attachments/assets/0f8f1065-26df-4cdd-b473-a538254788c2" />
<img width="1920" height="1080" alt="31" src="https://github.com/user-attachments/assets/fb056118-a32b-4423-85bd-35d38d5a9faf" />
<img width="1920" height="1080" alt="30" src="https://github.com/user-attachments/assets/5672b6c4-648a-4a86-a452-a884785c1987" />
<img width="1920" height="1080" alt="29" src="https://github.com/user-attachments/assets/21321099-3c3d-41e1-88e2-d0562f8344e0" />
<img width="1920" height="1080" alt="28" src="https://github.com/user-attachments/assets/6cf96698-8053-4a87-92c3-ce3e183dc00f" />
<img width="1920" height="1080" alt="27" src="https://github.com/user-attachments/assets/f436f2b3-e150-42b8-8341-613888afe635" />
<img width="1920" height="1080" alt="26" src="https://github.com/user-attachments/assets/f4201586-637f-46f1-bf28-aafe221f03bc" />
<img width="1920" height="1080" alt="25" src="https://github.com/user-attachments/assets/76e10fd4-779a-47e5-81b9-afbf7f5ea964" />
<img width="1920" height="1080" alt="24" src="https://github.com/user-attachments/assets/a7e6541b-ded7-4339-a8ac-e2057dd8d025" />
<img width="1920" height="1080" alt="23" src="https://github.com/user-attachments/assets/5756f5f5-9a5c-4d78-bba4-14b2d44f9ec7" />
<img width="1920" height="1080" alt="22" src="https://github.com/user-attachments/assets/c45d001a-b7a8-4832-aab8-cf59cd29639a" />
<img width="1920" height="1080" alt="21" src="https://github.com/user-attachments/assets/054adc19-ee5a-4b0c-b84c-5244b8b5cc75" />
<img width="1920" height="1080" alt="20" src="https://github.com/user-attachments/assets/7869d15c-7dee-489d-a333-f1b916b536d3" />
<img width="1920" height="1080" alt="19" src="https://github.com/user-attachments/assets/d03362cb-8d00-4985-9889-a6ab112ae7dc" />
<img width="1920" height="1080" alt="18" src="https://github.com/user-attachments/assets/00838247-f5b0-4897-8570-9991f4ea213b" />
<img width="1920" height="1080" alt="17" src="https://github.com/user-attachments/assets/cf1e77a5-b850-41c8-bd2a-52c188f2b49c" />
<img width="1920" height="1080" alt="16" src="https://github.com/user-attachments/assets/0d6fad47-33a3-43b0-8a96-74e536e12cb1" />
<img width="1920" height="1080" alt="15" src="https://github.com/user-attachments/assets/896a912f-172c-4746-bff1-e4e22b4606d0" />
<img width="1920" height="1080" alt="14" src="https://github.com/user-attachments/assets/5da174b9-aca3-4036-8879-1429710e8b9b" />
<img width="1920" height="1080" alt="13" src="https://github.com/user-attachments/assets/7d4e0482-6708-4a94-aaaf-179af3663f75" />
<img width="1920" height="1080" alt="12" src="https://github.com/user-attachments/assets/f581bf22-1e10-4042-84c6-ab234f5ded64" />
<img width="1920" height="1080" alt="11" src="https://github.com/user-attachments/assets/e8b48e3c-55b4-438f-8df1-e01534fce204" />
<img width="1920" height="1080" alt="10" src="https://github.com/user-attachments/assets/5ad6cdbf-f8f8-44b0-85e2-36a271c4e2fd" />
<img width="1920" height="1080" alt="9" src="https://github.com/user-attachments/assets/e7d816c0-2dbc-4c5c-8076-5946449f76d8" />
<img width="1920" height="1080" alt="8" src="https://github.com/user-attachments/assets/c0b31254-213f-4910-b4c9-2f7b652c7f14" />
<img width="1920" height="1080" alt="7" src="https://github.com/user-attachments/assets/413b6e37-a397-428e-a606-664adf968252" />
<img width="1920" height="1080" alt="6" src="https://github.com/user-attachments/assets/248b9c2d-be70-4d21-9136-999b036969db" />
<img width="1920" height="1080" alt="5" src="https://github.com/user-attachments/assets/16487a17-0ede-4b04-9b1c-e415740fcbf9" />
<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/922a0a38-4521-4beb-8c5d-40737e8d91a9" />
<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/f1aac219-1e23-4db3-9876-90a9f1cff187" />
<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/dfd9d9fb-20d3-4013-afff-34f31821182d" />
<img width="1920" height="1080" alt="01" src="https://github.com/user-attachments/assets/a4222b86-c0d7-4a53-8bef-5b62c9b888da" />
 
 
