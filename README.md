# Jenkins CI/CD with Docker

Two simple Jenkins pipelines that take a static HTML page from GitHub to a running Docker container on an AWS EC2 server.

- **CI pipeline:** checks out this repo and builds a Docker image.
- **CD pipeline:** runs a container from that image on the same Jenkins server.

Both pipelines are defined as code in Jenkinsfiles in this repo.

## Architecture

```
CI:  GitHub repo  -->  Jenkins ci-pipeline  -->  Docker image (my-image:latest)

CD:  Jenkins cd-pipeline  -->  Container (jenkins-practice-demo)  -->  Browser on port 8082
```

The CD job reuses the image built by the CI job. Both run on the same server, so no registry is needed.

## Repository structure

```
.
├── Dockerfile
├── index.html
├── jenkins/
│   ├── ci/
│   │   └── jenkinsfile
│   └── cd/
│       └── jenkinsfile
└── README.md
```

| File | Purpose |
|---|---|
| `index.html` | The static page being deployed |
| `Dockerfile` | Builds an nginx image that serves `index.html` |
| `jenkins/ci/jenkinsfile` | CI pipeline: checkout, then `docker build` |
| `jenkins/cd/jenkinsfile` | CD pipeline: replace the old container, run the new one |

## Pipelines

### CI (`jenkins/ci/jenkinsfile`)

| Stage | What it does |
|---|---|
| Git Checkout | Pulls the latest code from the `main` branch |
| Docker Image Build | Runs `docker build -t my-image:latest .` |

### CD (`jenkins/cd/jenkinsfile`)

| Stage | What it does |
|---|---|
| Stop Existing Container | Stops and removes `jenkins-practice-demo` if it exists |
| Start Container on Port 8082 | Runs `docker run -d --name jenkins-practice-demo -p 8082:80 my-image:latest` |

## Prerequisites

- An Ubuntu server (for example, an AWS EC2 instance) with Jenkins installed
- Docker installed on the same server
- The `jenkins` user allowed to run Docker
- Ports **8080** (Jenkins) and **8082** (the app) open in the security group

### Install Docker and give Jenkins access

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker

sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Check the permission:

```bash
sudo -u jenkins docker ps
```

## Jenkins setup

Create two **Pipeline** jobs with these settings.

| Setting | CI job | CD job |
|---|---|---|
| Definition | Pipeline script from SCM | Pipeline script from SCM |
| SCM | Git | Git |
| Repository URL | this repo's URL | this repo's URL |
| Branch Specifier | `*/main` | `*/main` |
| Script Path | `jenkins/ci/jenkinsfile` | `jenkins/cd/jenkinsfile` |

The Script Path is case-sensitive and must match the file name exactly.

## Usage

1. Push your changes to the `main` branch on GitHub.
2. Run the **CI** job and wait for a green build.
3. Run the **CD** job.
4. Open the app at `http://<server-public-ip>:8082`. Use `http`, not `https`.

Verify on the server:

```bash
docker images my-image
docker ps
curl http://localhost:8082
```

To redeploy after editing `index.html`: push, run CI, then run CD.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Build is green but has no stages | The Jenkinsfile on GitHub is empty or not pushed | Commit and push, and check the branch and Script Path |
| `Unable to find ... from git` | Script Path is wrong or has a typo | Use exactly `jenkins/ci/jenkinsfile` or `jenkins/cd/jenkinsfile` |
| `docker: not found` | Docker is not installed on the Jenkins server | Install Docker (see Prerequisites) |
| `permission denied ... docker.sock` | `jenkins` user is not in the `docker` group | `sudo usermod -aG docker jenkins`, then restart Jenkins |
| Page loads with `curl` but times out in the browser | Port 8082 is blocked in the AWS security group | Add an inbound TCP rule for port 8082 |

## What I learned

- A green build does not mean the pipeline did anything, so read the console output.
- Jenkins only sees what is on GitHub, not what is on your computer.
- Small configuration details cause most CI/CD problems.

## Next steps

- Trigger builds automatically on every push
- Push images to a registry such as Docker Hub
- Chain the CD job to run after CI succeeds
