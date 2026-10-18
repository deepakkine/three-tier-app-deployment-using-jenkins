












# Three-Tier Application Deployment on AWS EKS with Jenkins CI/CD

A three-tier web application deployed on Amazon EKS using Docker, Amazon ECR, Kubernetes, and Jenkins. This project demonstrates both **manual deployment** and **automated CI/CD deployment** triggered by GitHub pushes.

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [AWS Environment](#aws-environment)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Method 1: Manual Deployment](#method-1-manual-deployment)
- [Method 2: Automated Deployment with Jenkins](#method-2-automated-deployment-with-jenkins)
- [Backend Docker Image Optimization](#backend-docker-image-optimization)
- [Verification and Troubleshooting](#verification-and-troubleshooting)
- [Rollback](#rollback)
- [Security Notes](#security-notes)

## Project Overview

This project deploys a three-tier task management application consisting of:

- **Frontend:** React application served by Nginx.
- **Backend:** Node.js and Express REST API.
- **Database:** MongoDB for task persistence.

The application runs on Amazon EKS. Docker images are stored in private Amazon ECR repositories, and an AWS Application Load Balancer provides external HTTP access to the application.

The project supports two deployment workflows:

1. **Manual:** Build images, push them to ECR, and update Kubernetes deployments using `kubectl`.
2. **Automated:** Push changes to GitHub, trigger Jenkins through a webhook, build and push images, and deploy the new image versions to EKS.

## Architecture

```text
Developer
    |
    | Git push
    v
GitHub Repository
    |
    | Webhook
    v
Jenkins on Amazon EC2
    |
    | Build Docker images
    | Tag with Git commit SHA
    v
Amazon ECR
    |
    | Pull images
    v
Amazon EKS
    |
    +---- Frontend Deployment (React + Nginx)
    |
    +---- Backend Deployment (Node.js API)
    |
    +---- MongoDB Deployment
    |
    v
AWS Application Load Balancer
    |
    v
User's Browser
```

The frontend sends API requests through the load balancer to the backend. The backend connects to MongoDB using Kubernetes service discovery and environment-based configuration.

## Technology Stack

| Component | Technology |
|---|---|
| Cloud platform | AWS |
| Containerization | Docker |
| Container registry | Amazon ECR |
| Kubernetes platform | Amazon EKS |
| CI/CD | Jenkins |
| Source control | Git and GitHub |
| Frontend | React and Nginx |
| Backend | Node.js and Express |
| Database | MongoDB |
| External access | AWS Application Load Balancer |
| Command-line tools | AWS CLI and kubectl |

## AWS Environment

The current project configuration uses:

| Setting | Value |
|---|---|
| AWS Region | `ap-south-1` |
| EKS cluster | `three-tier-eks` |
| Kubernetes namespace | `three-tier` |
| Backend deployment | `api` |
| Frontend deployment | `frontend` |
| Database deployment | `mongodb` |
| Backend port | `3500` |
| Frontend service port | `80` |

Amazon ECR repositories:

```text
848504403730.dkr.ecr.ap-south-1.amazonaws.com/three-tier-backend

848504403730.dkr.ecr.ap-south-1.amazonaws.com/three-tier-frontend
```

Replace account IDs, cluster names, or resource names if you recreate the infrastructure in another AWS account.

## Repository Structure

```text
.
├── Application-Code/
│   ├── backend/
│   │   ├── Dockerfile
│   │   ├── .dockerignore
│   │   ├── package.json
│   │   ├── package-lock.json
│   │   ├── index.js
│   │   ├── db.js
│   │   ├── models/
│   │   └── routes/
│   └── frontend/
│       └── Dockerfile
├── Kubernetes-Manifests-file/
│   ├── Backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── Frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── Database/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── ingress.yaml
├── Jenkinsfile
└── README.md
```

The Kubernetes directory may contain additional configuration files, such as secrets and storage manifests, depending on your setup.

## Prerequisites

Before deploying, make sure you have:

- An AWS account and the required IAM permissions.
- An existing EKS cluster with worker nodes in the `Ready` state.
- Docker, AWS CLI, kubectl, Git, and Jenkins configured as needed.
- Two private ECR repositories for the frontend and backend.
- Kubernetes deployments, services, MongoDB configuration, storage, and required secrets configured in the `three-tier` namespace.
- Working EKS access from the machine performing the deployment.

For Jenkins automation, the Jenkins EC2 instance must also have Docker, AWS CLI, kubectl, Git access, and the necessary permissions to push to ECR and update EKS deployments.

Do not store AWS credentials, database passwords, or Kubernetes secret values in this repository.

---

## Method 1: Manual Deployment

This method builds and publishes Docker images manually, then updates the Kubernetes deployments.

### Step 1: Clone the repository

```bash
git clone https://github.com/deepakkine/three-tier-app-deployment-using-jenkins.git

cd three-tier-app-deployment-using-jenkins
```

### Step 2: Configure AWS access

Configure AWS credentials using an approved method, such as an EC2 IAM role or an AWS CLI profile.

Check the current identity and region:

```bash
aws sts get-caller-identity

aws configure get region
```

If you are using an IAM role on EC2, you generally do not need to store long-lived AWS access keys on the instance.

### Step 3: Connect kubectl to EKS

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name three-tier-eks
```

Verify cluster access:

```bash
kubectl get nodes

kubectl get pods -n three-tier
```

### Step 4: Set the image tag and ECR registry

Use a Git commit-based tag so each build can be identified.

```bash
export AWS_REGION=ap-south-1

export ECR_REGISTRY=848504403730.dkr.ecr.ap-south-1.amazonaws.com

export IMAGE_TAG=$(git rev-parse --short=12 HEAD)

echo "$IMAGE_TAG"
```

### Step 5: Build the backend image

```bash
docker build \
  -t "$ECR_REGISTRY/three-tier-backend:$IMAGE_TAG" \
  Application-Code/backend
```

The backend Dockerfile uses Node.js Alpine and installs production dependencies only.

### Step 6: Build the frontend image

The frontend needs the correct backend API URL when its React production bundle is built.

```bash
export FRONTEND_BACKEND_URL="http://YOUR-LOAD-BALANCER-DNS/api/tasks"
```

Replace `YOUR-LOAD-BALANCER-DNS` with the actual DNS name of your load balancer.

Build the image:

```bash
docker build \
  --build-arg REACT_APP_BACKEND_URL="$FRONTEND_BACKEND_URL" \
  -t "$ECR_REGISTRY/three-tier-frontend:$IMAGE_TAG" \
  Application-Code/frontend
```

### Step 7: Authenticate to Amazon ECR

```bash
aws ecr get-login-password --region "$AWS_REGION" |
  docker login \
    --username AWS \
    --password-stdin "$ECR_REGISTRY"
```

### Step 8: Push both images to ECR

```bash
docker push "$ECR_REGISTRY/three-tier-backend:$IMAGE_TAG"

docker push "$ECR_REGISTRY/three-tier-frontend:$IMAGE_TAG"
```

### Step 9: Update the Kubernetes deployments

Update the backend:

```bash
kubectl set image deployment/api \
  api="$ECR_REGISTRY/three-tier-backend:$IMAGE_TAG" \
  -n three-tier
```

Update the frontend:

```bash
kubectl set image deployment/frontend \
  frontend="$ECR_REGISTRY/three-tier-frontend:$IMAGE_TAG" \
  -n three-tier
```

Wait for the rollouts:

```bash
kubectl rollout status deployment/api \
  -n three-tier --timeout=180s

kubectl rollout status deployment/frontend \
  -n three-tier --timeout=180s
```

Confirm that the pods are healthy:

```bash
kubectl get deployments -n three-tier

kubectl get pods -n three-tier
```

The MongoDB deployment is not changed by these commands.

---

## Method 2: Automated Deployment with Jenkins

The Jenkins pipeline automates the checkout, build, push, and deployment steps.

### CI/CD workflow

```text
Developer pushes code to GitHub
              |
              v
GitHub webhook triggers Jenkins
              |
              v
Checkout source code
              |
              v
Generate Git commit image tag
              |
              v
Build backend and frontend images
              |
              v
Authenticate to Amazon ECR
              |
              v
Push both images to ECR
              |
              v
Update frontend and backend deployments
              |
              v
Wait for Kubernetes rollouts
              |
              v
Pipeline succeeds or reports an error
```

### Step 1: Configure the Jenkins EC2 instance

Install and configure Jenkins, Docker, AWS CLI, kubectl, and Git on the Jenkins server.

The Jenkins execution user must be able to run the required commands.

Verify the tools:

```bash
docker --version

aws --version

kubectl version --client

git --version
```

Verify AWS access:

```bash
aws sts get-caller-identity
```

Verify EKS access using the Jenkins execution user:

```bash
kubectl get deployments -n three-tier
```

Use an IAM role or another approved credential mechanism. Grant only the permissions required to build and push images and update the intended Kubernetes resources.

### Step 2: Create the Jenkins pipeline job

In Jenkins:

1. Select **New Item**.
2. Enter the job name `three-tier-cicd`.
3. Select **Pipeline**.
4. Under Pipeline configuration, choose **Pipeline script from SCM**.
5. Select **Git**.
6. Configure the repository:

   ```text
   https://github.com/deepakkine/three-tier-app-deployment-using-jenkins.git
   ```

7. Set the branch to:

   ```text
   */main
   ```

8. Set the script path to:

   ```text
   Jenkinsfile
   ```

9. Save the job.

### Step 3: Configure the GitHub webhook

In the GitHub repository, open **Settings → Webhooks → Add webhook**.

Configure the webhook URL using your Jenkins public URL:

```text
http://YOUR-JENKINS-URL:8080/github-webhook/
```

Select JSON as the content type and enable push events.

In Jenkins, enable **GitHub hook trigger for GITScm polling** in the job's build triggers.

For a production deployment, use HTTPS and restrict access to Jenkins. Do not expose the Jenkins administration interface broadly to the public internet.

### Step 4: Understand the Jenkinsfile stages

The current `Jenkinsfile` defines these stages:

**Checkout**

- Retrieves the source code from SCM.
- Generates an image tag from the current Git commit.

**Build Images**

- Builds the backend image from `Application-Code/backend`.
- Builds the frontend image from `Application-Code/frontend`.
- Supplies the frontend backend API URL as a build argument.

**Push Images to ECR**

- Authenticates to Amazon ECR.
- Pushes both images using the commit-based tag.

**Deploy to EKS**

- Updates the `api` deployment.
- Updates the `frontend` deployment.
- Waits for both rollouts to complete.

**Post**

- Logs out of ECR after the pipeline finishes.

### Step 5: Trigger an automated deployment

Make a change to the application or its configuration.

For example, update a backend file and commit it:

```bash
git add Application-Code/backend/Dockerfile \
        Application-Code/backend/.dockerignore

git commit -m "Optimize backend Docker image"
```

Push the commit:

```bash
git push origin main
```

The GitHub webhook should trigger the Jenkins pipeline automatically.

Open the `three-tier-cicd` job in Jenkins and inspect **Console Output**.

A successful run should show that:

- Source code was checked out.
- Both images were built.
- Both images were pushed to ECR.
- The EKS deployment images were updated.
- The backend and frontend rollouts completed successfully.

The image tag is derived from the Git commit, making it easier to identify which version was deployed.

---

## Backend Docker Image Optimization

The original backend image used Node.js 14 and installed dependencies with `npm install`.

The optimized Dockerfile uses a smaller Alpine-based Node.js image, installs only production dependencies, and runs as a non-root user.

Current backend Dockerfile:

```dockerfile
FROM node:20-alpine

ENV NODE_ENV=production
WORKDIR /usr/src/app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY db.js index.js ./
COPY models ./models
COPY routes ./routes

USER node

EXPOSE 3500

CMD ["node", "index.js"]
```

The backend `.dockerignore` excludes local dependencies, Git metadata, environment files, logs, and coverage output from the build context.

### Image-size comparison

| Image | Reported disk usage | Reported content size |
|---|---:|---:|
| Original backend image | 1.37 GB | 354 MB |
| Optimized backend image | 209 MB | 50.5 MB |

These figures were observed during the local Docker build. Actual image storage can vary depending on Docker version, shared layers, and platform.

The optimized image successfully started and returned HTTP 200 from the `/ok` endpoint during local testing.

The full EKS deployment must also be verified to confirm that the application connects to the real MongoDB service and handles application requests correctly.

**Note:** The build reported npm dependency vulnerabilities. Image-size optimization does not resolve those vulnerabilities; review and update dependencies separately, with regression testing.

---

## Verification and Troubleshooting

### Check Kubernetes resources

```bash
kubectl get all -n three-tier
```

### Check backend health

The backend exposes an `/ok` health endpoint.

From a machine that can reach the backend service or load balancer:

```bash
curl -i http://YOUR-LOAD-BALANCER-DNS/ok
```

Use the correct route for your ingress configuration. If `/ok` is not routed through the load balancer, test it through the backend service or port-forward instead.

### Check application logs

```bash
kubectl logs deployment/api -n three-tier

kubectl logs deployment/frontend -n three-tier
```

### Check rollout status

```bash
kubectl rollout status deployment/api -n three-tier

kubectl rollout status deployment/frontend -n three-tier
```

### Inspect a failing pod

```bash
kubectl get pods -n three-tier

kubectl describe pod POD_NAME -n three-tier
```

Replace `POD_NAME` with the actual pod name.

### Common problems

| Problem | What to check |
|---|---|
| ECR authentication failure | AWS identity, ECR permissions, and region |
| Image pull failure | Repository name, tag, node permissions, and image pull secret |
| Jenkins cannot access EKS | kubeconfig and permissions for the Jenkins execution user |
| Backend cannot connect to MongoDB | `MONGO_CONN_STR`, MongoDB service, and Kubernetes secrets |
| Frontend cannot call the API | API URL baked into the frontend build and load balancer routing |
| Rollout fails | Pod events, readiness probes, image tag, and container logs |
| GitHub push does not trigger Jenkins | Webhook delivery status and Jenkins job trigger configuration |

---

## Rollback

If a deployment fails, inspect the deployment history:

```bash
kubectl rollout history deployment/api -n three-tier

kubectl rollout history deployment/frontend -n three-tier
```

Roll back the backend:

```bash
kubectl rollout undo deployment/api -n three-tier
```

Roll back the frontend:

```bash
kubectl rollout undo deployment/frontend -n three-tier
```

Wait for the rollouts to complete and verify application health.

Keep older image tags in ECR according to your rollback and retention policy. Do not delete images that are still needed for recovery.

---

## Security Notes

- Do not commit AWS access keys, database credentials, `.env` files, or Kubernetes secret values.
- Use IAM roles and least-privilege permissions for Jenkins and EKS.
- Protect the Jenkins interface with HTTPS and restrict administrative access.
- Keep container images and dependencies updated.
- Use a secure secret-management mechanism for MongoDB credentials.
- Apply appropriate network access restrictions to EC2, Jenkins, EKS, and the load balancer.
- Review ECR lifecycle policies before removing old image tags.
- Test deployments and rollback procedures before using this setup for production workloads.

## Project Outcome

This project demonstrates a three-tier application running on Amazon EKS and two ways to deploy it:

- **Manual deployment:** Build, push, and update Kubernetes deployments using AWS CLI, Docker, ECR, and kubectl.
- **Automated CI/CD:** Use GitHub webhooks and Jenkins to build commit-tagged images, push them to ECR, and deploy them to EKS.

It also demonstrates Docker image optimization, Kubernetes rollout verification, and basic deployment troubleshooting.



