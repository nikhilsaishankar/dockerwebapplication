# dockerwebapp

A containerized Java web application (`vprofile`) backed by MySQL, packaged with
Docker and deployed across a Docker Swarm cluster.

---

## Advanced Deployment

![CI/CD](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?logo=jenkins&logoColor=white)
![Build](https://img.shields.io/badge/build-Maven-C71A36?logo=apachemaven&logoColor=white)
![Quality Gate](https://img.shields.io/badge/quality-SonarQube-4E9BCD?logo=sonarqube&logoColor=white)
![Artifacts](https://img.shields.io/badge/artifacts-Nexus-1B1C30?logo=sonatype&logoColor=white)
![Security](https://img.shields.io/badge/scan-Trivy-1904DA?logo=aqua&logoColor=white)
![Registry](https://img.shields.io/badge/registry-Docker%20Hub-2496ED?logo=docker&logoColor=white)
![Orchestration](https://img.shields.io/badge/orchestration-Docker%20Swarm-2496ED?logo=docker&logoColor=white)

This section documents the end-to-end CI/CD pipeline used to build, validate,
scan, and deploy the application. The pipeline takes source code through static
analysis, artifact publishing, image hardening, and a high-availability rollout
onto a multi-node Docker Swarm cluster.

### Pipeline Overview

The workflow is orchestrated by **Jenkins** running on a dedicated build agent,
and progresses through eight stages — from source checkout to a replicated Swarm
deployment.

```
 ┌──────────────┐
 │   Developer  │  git push
 │   (GitHub)   │────────────────┐
 └──────────────┘                │
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │                    JENKINS CI SERVER  (slave: Project-1)                │
 │                                                                         │
 │  1. Checkout ──► 2. SonarQube ──► 3. Maven Build ──► 4. Nexus Upload    │
 │     (SCM)           (analysis)        (mvn package)     (WAR artifact)   │
 │        │                                   │                            │
 │        └──────────────► 5. Docker Build ◄──┘                            │
 │                              │                                          │
 │                              ▼                                          │
 │                       6. Trivy Scan (vulnerability gate)                │
 │                              │                                          │
 │                              ▼                                          │
 │                       7. docker push ──► Docker Hub                     │
 └───────────────────────────────────────────────┬───────────────────────┘
                                                  │ 8. docker stack deploy
                                                  ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │                      DOCKER SWARM CLUSTER (overlay net)                  │
 │                                                                         │
 │   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐             │
 │   │  Manager    │      │  Worker 1   │      │  Worker 2   │             │
 │   │             │      │             │      │             │             │
 │   │  app  x N   │      │  app  x N   │      │  app  x N   │  :7777→8080 │
 │   │  db   x N   │      │  db   x N   │      │  db   x N   │  :3306      │
 │   └─────────────┘      └─────────────┘      └─────────────┘             │
 │         3 replicas per service · self-healing · load-balanced           │
 └───────────────────────────────────────────────────────────────────────┘
```

### Stage 1 — Source Code Management

The web application and database sources (each with its own `Dockerfile`) are
checked out from GitHub onto the Jenkins CI server. The pipeline is pinned to a
dedicated build agent (`Project-1`) so that builds run in an isolated,
reproducible environment rather than on the Jenkins controller.

```groovy
pipeline {
    agent { label 'Project-1' }   // dedicated slave node
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/<owner>/dockerwebapplication.git'
            }
        }
    }
}
```

The supporting infrastructure runs on AWS EC2 — a Jenkins controller, the Swarm
manager, a Nexus host, and two worker nodes:

![EC2 cluster instances](docs/images/img1.png)

In Jenkins, the build is pinned to the dedicated `Project-1` agent rather than
the built-in controller node:

![Jenkins nodes — Project-1 slave](docs/images/img2.png)

### Stage 2 — Code Quality Analysis (SonarQube)

Static code analysis is performed with **SonarQube** to enforce quality gates
before any artifact is built.

1. Install the **SonarQube Scanner** plugin in Jenkins.
2. Register the SonarQube server and authentication token under
   *Manage Jenkins → System → SonarQube servers*.
3. Wrap the analysis stage in `withSonarQubeEnv` so Jenkins injects the server
   URL and credentials automatically.

```groovy
stage('SonarQube Analysis') {
    environment {
        scannerHome = tool 'sonar-scanner'
    }
    steps {
        withSonarQubeEnv('sonar-server') {
            sh '''
                ${scannerHome}/bin/sonar-scanner \
                  -Dsonar.projectKey=vprofile \
                  -Dsonar.sources=src/ \
                  -Dsonar.java.binaries=target/classes
            '''
        }
    }
}
```

Analysis results — code smells, bugs, coverage, and the quality gate status —
are published to the SonarQube dashboard:

![SonarQube quality gate dashboard](docs/images/img3.png)

Individual issues (bugs, vulnerabilities, and security hotspots) can be drilled
into directly against the source:

![SonarQube issues detail](docs/images/img3.1.png)

### Stage 3 — Build (Maven)

The Maven tool is configured in Jenkins (*Manage Jenkins → Tools → Maven*) and
used to compile, test, and package the application into a deployable WAR.

```groovy
stage('Build') {
    steps {
        sh 'mvn clean package'
    }
}
```

```bash
# Produces the deployable artifact:
target/vprofile-v2.war
```

The Jenkins console confirms a successful Maven run with the WAR packaged:

![Jenkins Maven build success](docs/images/img4.png)

### Stage 4 — Artifact Registry (Nexus)

The packaged WAR is published to a **Nexus Repository Manager**, providing
versioned, immutable artifact storage and a single source of truth for releases.

```groovy
stage('Upload Artifact') {
    steps {
        nexusArtifactUploader(
            nexusVersion: 'nexus3',
            protocol: 'http',
            nexusUrl: '<nexus-host>:8081',
            groupId: 'com.visualpathit',
            version: "${BUILD_ID}",
            repository: 'vprofile-release',
            credentialsId: 'nexus-login',
            artifacts: [[
                artifactId: 'vprofile',
                file: 'target/vprofile-v2.war',
                type: 'war'
            ]]
        )
    }
}
```

The uploaded artifact, with its checksums, is then browsable in Nexus:

![Nexus repository — vprofile WAR](docs/images/img5.png)

### Stage 5 — Docker Image Build

Application and database images are built from the `Dockerfile`s tracked in the
repository. The web image layers the WAR onto Tomcat; the database image seeds
MySQL with the `accounts` schema.

```bash
# Application image (Tomcat 8 / JRE 11, ROOT.war)
docker build -t nikhilkotharu/dockerapp:web ./Docker-app

# Database image (MySQL 5.7.25, accounts DB)
docker build -t nikhilkotharu/dockerapp:db  ./Docker-db
```

### Stage 6 — Image Security Scan (Trivy)

**Trivy** is installed on the Swarm manager node and scans the freshly built
images for OS-package and application-dependency vulnerabilities, acting as a
security gate before publishing.

```bash
# Install Trivy on the manager node
sudo apt-get install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy

# Scan images — fail the build on HIGH/CRITICAL findings
trivy image --severity HIGH,CRITICAL nikhilkotharu/dockerapp:web
trivy image --severity HIGH,CRITICAL nikhilkotharu/dockerapp:db
```

### Stage 7 — Push to Registry (Docker Hub)

Once images clear the Trivy gate, they are pushed to **Docker Hub** so every
Swarm node can pull identical, scanned artifacts.

```bash
docker login -u nikhilkotharu
docker push nikhilkotharu/dockerapp:web
docker push nikhilkotharu/dockerapp:db
```

Both tags (`web` and `db`) land in the Docker Hub repository, ready for the
Swarm nodes to pull:

![Docker Hub repository tags](docs/images/img6.png)

### Stage 8 — Deploy with Docker Stack

The application is rolled out as a **Docker Stack** defined in
[`compose.yml`](./compose.yml). A stack is the bridge between the two worlds:
it consumes a standard Docker **Compose** file but schedules each service as a
replicated, cluster-aware **Swarm** workload — so a single file describes both
*what* to run and *how* to run it at scale.

#### Understanding `compose.yml`

The file declares two services, one network, and two named volumes:

```yaml
version: "3.9"
services:
  devopsdb:                              # MySQL database service
    image: nikhilkotharu/dockerapp:db
    ports:
      - "3306:3306"                      # expose MySQL on the cluster
    deploy:
      replicas: 3                        # 3 tasks for availability
      restart_policy:
        condition: any                   # self-heal — always restart
      resources:
        limits:    { cpus: '0.3', memory: '256m' }   # hard ceiling
        reservations: { cpus: '0.1', memory: '128m' } # guaranteed minimum
    networks:
      - mynetwork
    volumes:
      - db_volume:/opt/nikhil            # persist data across restarts

  application:                           # Java/Tomcat web service
    image: nikhilkotharu/dockerapp:web
    ports:
      - "7777:8080"                      # host 7777 → Tomcat 8080
    deploy:
      replicas: 3
      restart_policy:
        condition: any
      resources:
        limits:    { cpus: '0.3', memory: '256m' }
        reservations: { cpus: '0.1', memory: '128m' }
    depends_on:
      - devopsdb                         # start after the database
    networks:
      - mynetwork
    volumes:
      - app_volume:/opt/nikhil

networks:
  mynetwork:
    driver: overlay                      # multi-host networking across nodes

volumes:
  db_volume:
  app_volume:
```

Key directives:

| Directive | What it does |
|-----------|--------------|
| `deploy.replicas: 3` | Runs 3 tasks per service, load-balanced via the routing mesh |
| `restart_policy.condition: any` | Self-healing — failed tasks are always rescheduled |
| `resources.limits` / `reservations` | Caps and guarantees CPU/memory per task |
| `networks.driver: overlay` | Multi-host network so containers talk across nodes |
| `volumes` | Named volumes keep data persistent across task restarts |
| `depends_on` | Ensures the database service starts before the application |

#### Deploy commands

```bash
# Initialize the cluster (manager node, one-time)
docker swarm init

# Join workers using the token printed by `docker swarm init`
docker swarm join --token <worker-token> <manager-ip>:2377

# Deploy / update the stack from the compose file
docker stack deploy -c compose.yml stackname

# Verify the rollout
docker stack services stackname
docker service ps stackname_application
```

| Service       | Image                          | Replicas | Port (host → container) |
|---------------|--------------------------------|:--------:|-------------------------|
| `application` | `nikhilkotharu/dockerapp:web`  |    3     | `7777 → 8080` (Tomcat)  |
| `devopsdb`    | `nikhilkotharu/dockerapp:db`   |    3     | `3306 → 3306` (MySQL)   |

Running multiple replicas across multiple nodes provides **high availability**:
Swarm load-balances traffic via the routing mesh and reschedules tasks onto
healthy nodes if a container or node fails.

Once the stack is up, the application is reachable on port `7777` and serves the
login and registration pages:

![Deployed application — login page](docs/images/img7.png)

![Deployed application — sign-up page](docs/images/img7.1.png)

After authenticating, the user lands on the live profile feed — confirming the
full web ↔ database round-trip is working end to end:

![Deployed application — authenticated profile](docs/images/img9.png)

### Pipeline & Build Statistics

The Jenkins **Stage View** breaks down per-stage timing for a single run across
all eight stages (tool install → code → CQA → build → registry → docker images
→ image scan → registry → deploy):

![Jenkins Stage View — single run](docs/images/img8.png)

Build history across successive runs shows stage stability and trend over time:

![Jenkins Stage View — build history](docs/images/img8.1.png)

The post-build SonarQube quality gate result and build permalinks are surfaced
back in Jenkins:

![Jenkins — SonarQube quality gate & permalinks](docs/images/img8.2.png)

At the infrastructure level, AWS CloudWatch tracks CPU and network utilization
across all five cluster nodes:

![CloudWatch — cluster node metrics](docs/images/img9.1.png)

### Container-to-Container Communication (Overlay Network)

Because every service is attached to the `mynetwork` **overlay** network,
containers communicate across physical nodes as if on a single L2 segment.
The ping below succeeds between container IPs (`10.0.7.x`) scheduled on
different hosts — proving the overlay mesh is routing inter-container traffic:

![Overlay network — container-to-container ping](docs/images/img10.png)

### Database Verification

Connecting to the MySQL service confirms the `accounts` database and its seeded
tables were initialized correctly, with `SELECT` returning the expected rows:

![MySQL — database and table verification](docs/images/img11.png)

#### Tooling Summary

| Stage | Tool        | Purpose                                  |
|-------|-------------|------------------------------------------|
| 1     | Git / GitHub| Source code management                   |
| 2     | SonarQube   | Static analysis & quality gate           |
| 3     | Maven       | Compile, test, package WAR               |
| 4     | Nexus       | Artifact repository                      |
| 5     | Docker      | Image build                              |
| 6     | Trivy       | Image vulnerability scanning             |
| 7     | Docker Hub  | Container registry                       |
| 8     | Docker Swarm| Orchestration & high-availability deploy |