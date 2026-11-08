# DemoApp

A minimal **Spring Boot** application used as the target of a CI/CD and GitOps series of labs (iVolve Cloud DevOps Accelerator). It answers **`Hello from Dockerized Spring Boot!`** on port **8080**.

This repository is a fork of [`Ibrahim-Adel15/DemoApp`](https://github.com/Ibrahim-Adel15/DemoApp). Each lab lives on its own branch, so the pipeline of one lab never breaks another.

## Branches

| Branch | Lab | What it contains | Jenkins job |
|--------|-----|------------------|-------------|
| `main` | Lab 21: Jenkins Pipeline for Application Deployment | A 7-stage `Jenkinsfile` (build, image, scan, push, delete, update manifest, `kubectl apply`) and `deployment.yaml` in the root | `DemoApp` |
| `lab22` | Lab 22: Jenkins Agents and Shared Libraries | A short `Jenkinsfile` that loads the shared library and runs on the agent `agent-1` | `DemoApp-Lab22` |
| `lab23` | Lab 23: GitOps with Jenkins and Argo CD | The same `Jenkinsfile` as `lab22`, and the manifests moved to `k8s/`. Jenkins pushes the new image tag here and Argo CD syncs it to the cluster | `DemoApp-Lab23` |

## Repository layout

| Path | Purpose |
|------|---------|
| `src/main/java/com/example/demo/DemoApplication.java` | The Spring Boot application |
| `pom.xml` | Maven build (`mvn clean package` produces `target/demo-0.0.1-SNAPSHOT.jar`) |
| `Dockerfile` | Packages the jar into a container image (port 8080) |
| `Jenkinsfile` | The pipeline of the branch (differs per branch, see above) |
| `deployment.yaml` | The Deployment manifest (`main` and `lab22`) |
| `k8s/deployment.yaml`, `k8s/service.yaml` | The manifests watched by Argo CD (`lab23`) |

## Run it locally

```bash
mvn clean package
java -jar target/demo-0.0.1-SNAPSHOT.jar
curl http://localhost:8080/
```

```
Hello from Dockerized Spring Boot!
```

With Docker:

```bash
docker build -t nourz1/demo-app:local .
docker run -p 8080:8080 nourz1/demo-app:local
```

The pipeline publishes the images to Docker Hub as [`nourz1/demo-app`](https://hub.docker.com/r/nourz1/demo-app), tagged with the Jenkins build number.

## How the pipeline works

**Lab 21 (`main`)**: Jenkins builds the app, builds and scans the image with Trivy, pushes it to Docker Hub, removes the local image, edits the image tag in `deployment.yaml` and runs `kubectl apply`.

**Lab 22 (`lab22`)**: the same work is split into three shared-library steps (`buildApp`, `buildImage`, `deployOnK8s`) from [`jenkins-shared-library`](https://github.com/neiorz/jenkins-shared-library), and the pipeline runs on a Jenkins agent:

```groovy
@Library('my-shared-lib') _

pipeline {
    agent { label 'agent-1' }
    environment { IMAGE_NAME = 'nourz1/demo-app' }
    stages {
        stage('1. Build App')    { steps { buildApp() } }
        stage('2. Build Image')  { steps { buildImage(env.IMAGE_NAME, env.BUILD_NUMBER) } }
        stage('3. Deploy on K8s') { steps { deployOnK8s(env.IMAGE_NAME, env.BUILD_NUMBER) } }
    }
}
```

**Lab 23 (`lab23`)**: `deployOnK8s` no longer runs `kubectl`. It edits `k8s/deployment.yaml` and pushes the commit to this branch. An **Argo CD** Application named `ivolve` watches `lab23/k8s` with automatic sync and applies the change to the cluster.

> The `lab22` branch expects `deployment.yaml` in the repository root, while `lab23` expects `k8s/deployment.yaml`. The shared library's current `deployOnK8s` is the GitOps version (Lab 23).

## Related repositories

| Repository | Purpose |
|------------|---------|
| [`neiorz/jenkins-shared-library`](https://github.com/neiorz/jenkins-shared-library) | The Groovy shared library used by the pipelines |
| [`neiorz/iVolve-labs`](https://github.com/neiorz/iVolve-labs) | The full lab reports (Jenkins Lab 20-22 and ArgoCD Lab 23) |

## License

MIT, see [`LICENSE`](LICENSE).
