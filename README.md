# Three-Tier Application Deployment on AWS EKS with Jenkins CI/CD

A hands-on DevOps project that deploys a React frontend, Node.js/Express API, and MongoDB database on Amazon EKS. It demonstrates both **manual deployment** and a **GitHub → Jenkins → Amazon ECR → Amazon EKS** CI/CD workflow.

> **Important:** AWS resources incur charges. Create resources only in a test account, use the smallest suitable capacity, and delete resources when finished. This guide uses `ap-south-1` (Mumbai) as an example region. Replace account IDs, URLs, names, and any repository-specific values with your own.

## Table of contents

- [Architecture](#architecture)
- [Technology stack](#technology-stack)
- [Repository layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [1. Install tools on an Ubuntu workstation](#1-install-tools-on-an-ubuntu-workstation)
- [2. Configure AWS access](#2-configure-aws-access)
- [3. Create ECR repositories](#3-create-ecr-repositories)
- [4. Create the EKS cluster](#4-create-the-eks-cluster)
- [5. Deploy the application manually](#5-deploy-the-application-manually)
- [6. Set up Jenkins on EC2](#6-set-up-jenkins-on-ec2)
- [7. Configure Jenkins access to AWS and EKS](#7-configure-jenkins-access-to-aws-and-eks)
- [8. Configure the Jenkins pipeline](#8-configure-the-jenkins-pipeline)
- [9. Configure the GitHub webhook](#9-configure-the-github-webhook)
- [10. Verify the deployment](#10-verify-the-deployment)
- [Rollback](#rollback)
- [Troubleshooting](#troubleshooting)
- [Security recommendations](#security-recommendations)
- [Cleanup](#cleanup)
- [Project limitations and possible improvements](#project-limitations-and-possible-improvements)

## Architecture

```text
Developer
   |
   | git push
   v
GitHub repository
   | webhook
   v
Jenkins on Ubuntu EC2
   | checkout, build, tag images with Git commit SHA
   v
Amazon ECR (private repositories)
   | image pull
   v
Amazon EKS
   |-- frontend Deployment (React served by Nginx)
   |-- API Deployment (Node.js / Express)
   |-- MongoDB Deployment and Service
   v
AWS Load Balancer / Ingress
   |
   v
Browser
```

The frontend calls the API through the configured API URL. The API connects to MongoDB through Kubernetes service discovery and the database connection configuration supplied to the backend.

## Technology stack

| Layer | Technology |
|---|---|
| Cloud | AWS |
| Containers | Docker |
| Container registry | Amazon ECR |
| Kubernetes | Amazon EKS, `kubectl` |
| CI/CD | Jenkins |
| Source control and webhook | Git, GitHub |
| Frontend | React, Nginx |
| Backend | Node.js, Express |
| Database | MongoDB |
| External routing | AWS Load Balancer / Kubernetes ingress, depending on cluster configuration |

## Repository layout

The repository is expected to have a structure similar to the following. Check the actual files in your clone before running commands; filenames can differ between revisions.

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

If your manifests include Kubernetes `Secret` objects, persistent-volume claims, storage classes, or ingress-controller resources, review and configure those before deployment. Do not commit real secret values to Git.

## Prerequisites

- An AWS account with permission to create EKS, EC2, ECR, IAM, networking, and load-balancing resources.
- A GitHub account and this repository forked or copied into your own account.
- An Ubuntu workstation or Ubuntu EC2 instance for command-line operations.
- A public/private networking design that lets the workstation and Jenkins reach the EKS API endpoint.
- A supported Kubernetes version and compatible `kubectl` version.
- A working database configuration and Kubernetes manifests for the application.

**Cost warning:** EKS control planes, EC2 instances, NAT gateways, load balancers, EBS volumes, public IPv4 addresses, and stored images can incur charges. A stopped EC2 instance may still incur storage or address charges. Review the AWS pricing for your region before starting.

---

## 1. Install tools on an Ubuntu workstation

The commands below target a recent Ubuntu LTS system. Review official vendor instructions if your OS or CPU architecture differs.

### 1.1 Update the operating system and install utilities

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl unzip gnupg lsb-release git jq
```

### 1.2 Install AWS CLI v2

```bash
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install
aws --version
```

For ARM-based machines, download the AWS CLI package for `aarch64` instead of `x86_64`. If AWS CLI is already installed, consult the official upgrade instructions rather than repeatedly running the installer.

### 1.3 Install kubectl

This example downloads the stable release and verifies the binary architecture against the host. For production, choose a `kubectl` version compatible with your EKS cluster.

```bash
ARCH="$(dpkg --print-architecture)"
case "$ARCH" in
  amd64) KARCH="amd64" ;;
  arm64) KARCH="arm64" ;;
  *) echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac
KVER="$(curl -fsSL https://dl.k8s.io/release/stable.txt)"
curl -fsSLo /tmp/kubectl "https://dl.k8s.io/release/${KVER}/bin/linux/${KARCH}/kubectl"
curl -fsSLo /tmp/kubectl.sha256 "https://dl.k8s.io/release/${KVER}/bin/linux/${KARCH}/kubectl.sha256"
echo "$(cat /tmp/kubectl.sha256)  /tmp/kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 /tmp/kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 1.4 Install Docker Engine

Use Docker's official Ubuntu repository. These commands install Docker Engine and the CLI:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu ${UBUNTU_CODENAME:-$VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker version
```

To use Docker without `sudo`, add your account to the `docker` group and sign out/in:

```bash
sudo usermod -aG docker "$USER"
```

**Security note:** Membership in the `docker` group effectively grants root-level control of the host. Only grant it to trusted users. On Jenkins, configure Docker access deliberately and do not treat this group as a low-privilege role.

### 1.5 Install eksctl (optional, used to create EKS)

```bash
ARCH="$(dpkg --print-architecture)"
case "$ARCH" in
  amd64) EARCH="amd64" ;;
  arm64) EARCH="arm64" ;;
  *) echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac
curl -fsSL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_${EARCH}.tar.gz" -o /tmp/eksctl.tar.gz
tar -xzf /tmp/eksctl.tar.gz -C /tmp
sudo install -o root -g root -m 0755 /tmp/eksctl /usr/local/bin/eksctl
eksctl version
```

---

## 2. Configure AWS access

Use an IAM role where available. For a local workstation, an AWS IAM Identity Center profile or a dedicated short-lived role is preferable to long-lived access keys.

For an approved CLI profile, configure it interactively:

```bash
aws configure --profile devops-lab
export AWS_PROFILE=devops-lab
export AWS_REGION=ap-south-1
aws sts get-caller-identity
```

The identity command must succeed before continuing. Never paste access keys, secret keys, session tokens, kubeconfig credentials, or database passwords into this README or commit them to Git.

Set common project variables (replace the account ID with the one returned by your own AWS setup):

```bash
export AWS_REGION="ap-south-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
export CLUSTER_NAME="three-tier-eks"
export NAMESPACE="three-tier"
export IMAGE_TAG="$(git rev-parse --short=12 HEAD)"
printf 'Account: %s\nRegion: %s\nImage tag: %s\n' "$AWS_ACCOUNT_ID" "$AWS_REGION" "$IMAGE_TAG"
```

Use an IAM policy scoped to the required resources. Do not copy an unrestricted `AdministratorAccess` policy into a production setup.

## 3. Create ECR repositories

Create the two private repositories if they do not already exist:

```bash
aws ecr describe-repositories --region "$AWS_REGION" --repository-names three-tier-frontend three-tier-backend >/dev/null 2>&1 || true
aws ecr create-repository --region "$AWS_REGION" --repository-name three-tier-frontend 2>/dev/null || true
aws ecr create-repository --region "$AWS_REGION" --repository-name three-tier-backend 2>/dev/null || true
aws ecr describe-repositories --region "$AWS_REGION" --repository-names three-tier-frontend three-tier-backend
```

The `|| true` clauses make the creation attempt tolerant of an existing repository, but they can also hide other errors. Read the final `describe-repositories` output and correct any access or region errors before continuing.

Repository URIs will be:

```text
<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/three-tier-frontend
<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/three-tier-backend
```

## 4. Create the EKS cluster

If you already have an EKS cluster with worker nodes and the required IAM access, skip to the next section. Otherwise, the following `eksctl` example creates a small managed-node-group cluster. It can take several minutes and creates billable resources.

```bash
eksctl create cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --version 1.33 \
  --managed \
  --nodegroup-name app-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --with-oidc
```

Kubernetes versions supported by EKS change over time. If the specified version is no longer available, select a currently supported version from the AWS EKS console and use that version in the command.

Configure `kubectl`:

```bash
aws eks update-kubeconfig --region "$AWS_REGION" --name "$CLUSTER_NAME"
kubectl get nodes
```

All expected worker nodes should show `Ready`. If the cluster already exists, do not run the create command again. Make sure the caller has EKS cluster access and the required Kubernetes RBAC permissions.

Create the namespace if it does not exist:

```bash
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
```

### Prepare database configuration before applying manifests

Inspect the backend configuration and database manifests. The backend needs the correct MongoDB service hostname, database name, and credentials if authentication is enabled. Create Kubernetes Secrets using your own secure values or a secret manager. Do not place real passwords in source control or shell history.

Example pattern (replace the placeholder values securely):

```bash
kubectl create secret generic mongodb-credentials \
  --namespace "$NAMESPACE" \
  --from-literal=username='<DB_USERNAME>' \
  --from-literal=password='<DB_PASSWORD>'
```

Only use this example if the actual manifests and backend expect a Secret named `mongodb-credentials` with these keys. Align the name and keys with the code/manifests. For real deployments, prefer a managed secret solution and avoid typing passwords into shell history.

**Database storage:** A simple MongoDB Deployment without persistent storage can lose data when pods are replaced. For production or data you need to retain, configure a persistent volume or use a managed database such as Amazon DocumentDB or MongoDB Atlas. Verify storage, backup, and recovery requirements before deploying.

## 5. Deploy the application manually

From the repository root:

```bash
git clone https://github.com/deepakkine/three-tier-app-deployment-using-jenkins.git
cd three-tier-app-deployment-using-jenkins
```

If you have forked the repository, use your fork's URL. Confirm the repository structure and inspect the Dockerfiles/manifests before applying them.

### 5.1 Build the backend image

```bash
export AWS_REGION="ap-south-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
export IMAGE_TAG="$(git rev-parse --short=12 HEAD)"

docker build \
  -t "$ECR_REGISTRY/three-tier-backend:$IMAGE_TAG" \
  Application-Code/backend
```

The optimized backend Dockerfile in this project uses `node:20-alpine`, installs production dependencies with `npm ci --omit=dev`, copies only the required application files, and runs as the non-root `node` user. Check compatibility with your dependencies before changing the Node.js version.

### 5.2 Determine the frontend API URL and build the frontend

The React API URL is often embedded into the frontend bundle at build time. Use the actual load balancer/Ingress DNS name and path for your configuration:

```bash
export FRONTEND_BACKEND_URL="http://<YOUR-LOAD-BALANCER-DNS>/api/tasks"

docker build \
  --build-arg REACT_APP_BACKEND_URL="$FRONTEND_BACKEND_URL" \
  -t "$ECR_REGISTRY/three-tier-frontend:$IMAGE_TAG" \
  Application-Code/frontend
```

Replace the placeholder. Confirm the frontend Dockerfile actually consumes `REACT_APP_BACKEND_URL`; changing a build argument after the image is built will not change an existing image. Use HTTPS for a production-facing application and configure CORS/routing as needed.

### 5.3 Push images to ECR

```bash
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$ECR_REGISTRY"

docker push "$ECR_REGISTRY/three-tier-backend:$IMAGE_TAG"
docker push "$ECR_REGISTRY/three-tier-frontend:$IMAGE_TAG"
docker logout "$ECR_REGISTRY"
```

### 5.4 Apply Kubernetes manifests

Review manifests first, especially namespaces, image names, Secrets, storage, service selectors, ingress class, and public exposure. If all manifests are designed for this namespace and are safe to apply together:

```bash
kubectl apply -f Kubernetes-Manifests-file/Database/ -n "$NAMESPACE"
kubectl apply -f Kubernetes-Manifests-file/Backend/ -n "$NAMESPACE"
kubectl apply -f Kubernetes-Manifests-file/Frontend/ -n "$NAMESPACE"
```

Apply the ingress separately only after an ingress controller/load-balancer integration has been installed and the ingress manifest matches that controller:

```bash
kubectl apply -f Kubernetes-Manifests-file/ingress.yaml -n "$NAMESPACE"
```

If your manifests already specify a namespace, or contain objects that must be created outside `three-tier`, adapt the commands accordingly. If MongoDB is already deployed, applying its manifests may alter it; review the YAML before applying. Do not apply a directory containing example or backup manifests by accident.

### 5.5 Update deployments to use the new images

If your manifests use other deployment or container names, adjust these commands to match `kubectl get deployments -n "$NAMESPACE"` and the actual YAML:

```bash
kubectl set image deployment/api \
  api="$ECR_REGISTRY/three-tier-backend:$IMAGE_TAG" \
  -n "$NAMESPACE"

kubectl set image deployment/frontend \
  frontend="$ECR_REGISTRY/three-tier-frontend:$IMAGE_TAG" \
  -n "$NAMESPACE"

kubectl rollout status deployment/api -n "$NAMESPACE" --timeout=180s
kubectl rollout status deployment/frontend -n "$NAMESPACE" --timeout=180s
kubectl get deployments,pods,services -n "$NAMESPACE"
```

The manual image update changes the frontend and backend only; it does not automatically update MongoDB.

---

## 6. Set up Jenkins on EC2

Create an Ubuntu EC2 instance for Jenkins. Use a supported Ubuntu LTS AMI, a suitable instance size for your build workload, and a security group that restricts SSH to your own IP. Port `8080` should be restricted to trusted access or placed behind a secure reverse proxy with HTTPS. Do not expose Jenkins administration to the whole internet.

### 6.1 Install Java 21 and Jenkins

Jenkins package requirements change over time. Follow the official Jenkins Linux installation instructions if the repository or supported Java version changes. Example for a current Ubuntu system where Java 21 is supported:

```bash
sudo apt-get update
sudo apt-get install -y fontconfig openjdk-21-jre
java -version

sudo install -m 0755 -d /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
sudo sh -c 'echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

Get the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open `http://<JENKINS-EC2-PUBLIC-DNS>:8080` only from a trusted network, complete the setup wizard, install suggested plugins, and create an administrator account. For production, configure HTTPS and stronger access controls before exposing Jenkins to external users.

### 6.2 Install Docker on the Jenkins instance

Use the Docker Engine installation steps in [Install Docker Engine](#14-install-docker-engine) on the Jenkins EC2 host. Then configure access for Jenkins deliberately:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Docker group membership gives the Jenkins account powerful host-level access. A hardened alternative is to use isolated build agents or a rootless/build service design.

### 6.3 Install AWS CLI, kubectl, and Git on Jenkins

Install AWS CLI v2 and kubectl using the instructions in [Install tools on an Ubuntu workstation](#1-install-tools-on-an-ubuntu-workstation). Install Git if it is not already present:

```bash
sudo apt-get update
sudo apt-get install -y git
```

Verify the commands as the Jenkins execution user—not just your SSH login:

```bash
sudo -u jenkins -H bash -lc 'git --version; docker --version; aws --version; kubectl version --client'
```

If the Jenkins service runs with a restricted PATH, configure the tool locations explicitly in Jenkins or the service environment. Do not work around a broken system by repeatedly reinstalling binaries without checking the underlying error.

---

## 7. Configure Jenkins access to AWS and EKS

### 7.1 AWS permissions for Jenkins

Prefer attaching a dedicated IAM role to the Jenkins EC2 instance. The role needs narrowly scoped permissions to:

- Authenticate to ECR and push to the two project repositories.
- Describe the EKS cluster and obtain EKS authentication tokens.
- Access the Kubernetes API using a principal authorized by EKS access entries or the cluster's configured access mechanism.

ECR push permissions commonly include repository-scoped image upload actions and `ecr:GetAuthorizationToken` (which requires `Resource: "*"`). Exact permissions depend on your setup. Avoid attaching `AdministratorAccess` for a production pipeline.

Check the role from the Jenkins execution user:

```bash
sudo -u jenkins -H bash -lc 'aws sts get-caller-identity'
```

This should return the expected IAM role identity without requiring static AWS keys.

### 7.2 Configure EKS access

Run as the Jenkins user so the kubeconfig is written to the correct home directory:

```bash
sudo -u jenkins -H bash -lc 'aws eks update-kubeconfig --region ap-south-1 --name three-tier-eks'
sudo -u jenkins -H bash -lc 'kubectl get deployments -n three-tier'
```

The IAM role must be granted access to the EKS cluster and Kubernetes resources. Depending on the cluster, configure an EKS access entry and an appropriate access policy, or the supported Kubernetes RBAC mapping. Do not assume that AWS API permissions alone grant Kubernetes permissions.

For a production pipeline, limit Kubernetes permissions to the namespace and resources it needs rather than granting cluster-admin. Protect the kubeconfig and Jenkins credentials.

---

## 8. Configure the Jenkins pipeline

The included `Jenkinsfile` is expected to implement these stages:

1. **Checkout:** fetch the source from the configured Git branch and derive a 12-character image tag from the Git commit SHA.
2. **Build Images:** build backend and frontend images; pass the backend API URL as a frontend build argument.
3. **Push Images to ECR:** authenticate using `aws ecr get-login-password` and push both commit-tagged images.
4. **Deploy to EKS:** update the `api` and `frontend` Deployments and wait for rollout status.
5. **Post:** log out of ECR whether the pipeline succeeds or fails.

Check the repository's actual `Jenkinsfile` before relying on these stage names or environment values. The described pipeline does **not** automatically provision the EKS cluster or MongoDB, run a comprehensive test suite, scan images, perform a manual approval, or guarantee automatic rollback.

### Create the Jenkins job

1. In Jenkins, select **New Item**.
2. Name it `three-tier-cicd` and select **Pipeline**.
3. Choose **Pipeline script from SCM**.
4. Select **Git** and enter your repository URL, for example:

   ```text
   https://github.com/<YOUR_GITHUB_USER>/three-tier-app-deployment-using-jenkins.git
   ```

5. Set the branch to `*/main` (or your actual branch).
6. Set the script path to `Jenkinsfile`.
7. Save the job.
8. Under **Build Triggers**, enable **GitHub hook trigger for GITScm polling** if using the GitHub webhook.
9. Run **Build Now** once and review the console output before enabling automated triggers.

Ensure the Jenkins user can access the repository. For a public repository no Git credentials may be needed; for a private repository, configure credentials in Jenkins and reference the credential ID instead of embedding tokens in the Jenkinsfile.

### Configure pipeline environment values

The `Jenkinsfile` should use the correct values for:

- AWS region, e.g. `ap-south-1`.
- AWS account ID and ECR registry.
- Frontend and backend ECR repository names.
- Frontend backend API URL, including the correct scheme and route.
- Kubernetes namespace and deployment/container names.

Avoid hardcoding credentials. Non-secret configuration may be stored in the Jenkinsfile or job configuration; secrets belong in Jenkins Credentials or a managed secret service.

---

## 9. Configure the GitHub webhook

1. In GitHub, open the repository's **Settings → Webhooks → Add webhook**.
2. Set the payload URL to your reachable Jenkins endpoint:

   ```text
   https://<YOUR-SECURE-JENKINS-HOST>/github-webhook/
   ```

3. Choose `application/json` as the content type.
4. Select **Just the push event** unless other events are intentionally needed.
5. Configure a webhook secret and validate it in your integration if supported by your Jenkins/plugin configuration.
6. Add the webhook and inspect its **Recent Deliveries** for a successful response.
7. Confirm the Jenkins job's GitHub hook trigger is enabled.

A public HTTP URL such as `http://<PUBLIC-IP>:8080/github-webhook/` may work for a temporary lab but is not recommended for production. Use HTTPS, restrict inbound traffic, and do not expose the Jenkins administrative UI indiscriminately.

### Trigger a deployment

Make a change, commit it, and push to the configured branch:

```bash
git add Application-Code/backend/Dockerfile Application-Code/backend/.dockerignore
git commit -m "Optimize backend Docker image"
git push origin main
```

Use paths that you actually changed. The webhook should trigger Jenkins, which checks out the new commit, builds and pushes both images, and updates the application deployments. Monitor the build's Console Output and verify the rollouts afterward.

---

## 10. Verify the deployment

### Kubernetes status

```bash
kubectl get nodes
kubectl get all -n three-tier
kubectl get events -n three-tier --sort-by=.lastTimestamp
```

### Rollout and logs

```bash
kubectl rollout status deployment/api -n three-tier --timeout=180s
kubectl rollout status deployment/frontend -n three-tier --timeout=180s
kubectl logs deployment/api -n three-tier --tail=100
kubectl logs deployment/frontend -n three-tier --tail=100
```

### Check the application endpoint

The backend in this project exposes an `/ok` health endpoint. Use the actual URL/routing configured in your manifests:

```bash
curl -i "http://<YOUR-LOAD-BALANCER-DNS>/ok"
```

If `/ok` is not routed through the load balancer, test it through the Kubernetes Service or use `kubectl port-forward`. Also test the frontend in a browser, API create/read/update/delete operations as applicable, and persistence after a MongoDB pod restart if persistent storage is configured.

### Common problems

| Symptom | Checks |
|---|---|
| `aws sts get-caller-identity` fails | IAM role/profile, credential chain, instance metadata access, AWS CLI health, region |
| `kubectl` cannot get credentials | AWS CLI works as the same user, kubeconfig points to the right cluster, EKS access is configured |
| ECR login/push fails | Account ID, region, IAM push permissions, repository names |
| `ImagePullBackOff` | Image URI/tag, node ECR permissions, repository region, pod events |
| Jenkins works interactively but fails in a build | PATH, Jenkins user's IAM role, kubeconfig home directory, Docker access |
| GitHub push does not trigger Jenkins | Webhook delivery status, Jenkins URL/reachability, job trigger, branch and SCM configuration |
| API cannot connect to MongoDB | MongoDB Service DNS, Secret names/keys, connection string, network policy, pod logs |
| Frontend cannot call API | Frontend build-time API URL, HTTPS/mixed-content issues, CORS, ingress/load-balancer path rules |
| Rollout times out | `kubectl describe pod`, readiness/liveness probes, image logs, capacity and events |

---

## Rollback

Inspect deployment history:

```bash
kubectl rollout history deployment/api -n three-tier
kubectl rollout history deployment/frontend -n three-tier
```

Roll back to the previous ReplicaSet revision:

```bash
kubectl rollout undo deployment/api -n three-tier
kubectl rollout undo deployment/frontend -n three-tier
kubectl rollout status deployment/api -n three-tier --timeout=180s
kubectl rollout status deployment/frontend -n three-tier --timeout=180s
```

A Kubernetes rollback is only useful if the previous ReplicaSet and its image remain available. Keep older ECR tags according to a retention policy. Database schema or data changes may require a separate recovery plan.

## Security recommendations

- Prefer IAM roles and temporary credentials; never commit access keys or real secrets.
- Scope IAM permissions to the minimum actions and resources required.
- Keep Jenkins private or protected by HTTPS, network restrictions, authentication, and timely updates.
- Treat Docker access for Jenkins as privileged host access.
- Store database passwords in Kubernetes Secrets or a managed secret service; restrict who can read them.
- Do not expose MongoDB publicly.
- Use TLS for public application traffic and configure ingress/load-balancer security groups appropriately.
- Pin and update base images and dependencies; scan images and resolve dependency vulnerabilities.
- Add automated tests, image scanning, approvals where appropriate, and rollback/alerting procedures before production use.
- Configure ECR lifecycle policies and backup policies intentionally.

## Cleanup

Delete only resources created for this project, and verify the AWS region/account before every destructive action. Deleting the EKS cluster does not necessarily delete all related resources. Load balancers, EBS volumes, snapshots, Elastic IPs, NAT gateways, ECR repositories, EC2 instances, security groups, IAM roles, and CloudWatch logs may remain and may incur charges.

### Suggested cleanup order

1. Stop GitHub-triggered builds and disable the Jenkins job if it is no longer needed.
2. Delete the EKS managed node group(s), then delete the EKS cluster using the EKS console or `eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"`.
3. In EC2 Load Balancers, remove any project load balancers that remain after cluster deletion.
4. In ECR, delete the `three-tier-frontend` and `three-tier-backend` repositories if you no longer need the images.
5. Terminate the Jenkins EC2 instance after confirming you no longer need its data. Delete associated EBS volumes only if they are not needed.
6. Review VPC, NAT gateways, Elastic IPs, EBS volumes/snapshots, security groups, CloudWatch log groups, IAM roles/policies, and other project-created resources. Delete only resources you can positively identify as belonging to this project.
7. Review **AWS Billing → Bills/Cost Explorer** after cleanup. Billing data may take time to reflect resource deletion.

Do not delete shared VPCs, shared IAM roles, or resources used by other applications. If a resource is managed by CloudFormation, eksctl, or another IaC tool, prefer deleting it through its owning tool when possible.

## Project limitations and possible improvements

The current CI/CD flow automates application image build, push, and frontend/backend Deployment updates. The EKS cluster, MongoDB, and foundational Kubernetes resources may need to be created separately depending on the manifests and environment. Before production use, consider adding:

- Unit/integration tests and an explicit test stage.
- Image vulnerability scanning and dependency checks.
- A deployment approval stage for sensitive environments.
- Immutable image tags and ECR lifecycle policies.
- Health checks and automated rollback/notifications.
- Infrastructure as Code (Terraform, CloudFormation, or eksctl configuration) for reproducible infrastructure.
- Persistent database storage, backup/restore tests, and secrets management.
- Least-privilege IAM, private networking, HTTPS, and restricted Jenkins access.

## Project outcome

This project demonstrates two ways to release a three-tier application on AWS:

- **Manual deployment:** build and tag Docker images, push them to ECR, and update Kubernetes Deployments with `kubectl`.
- **Automated CI/CD:** GitHub push triggers Jenkins to build commit-tagged images, push them to ECR, and deploy frontend/backend updates to EKS.

It also demonstrates Kubernetes rollout verification, Docker image optimization, and basic troubleshooting. Verify the exact repository manifests and environment-specific values before using the commands in a new AWS account.
