# Jenkins CI/CD for a Dockerized Node App (EC2 Controller + Agent)

This project demonstrates a complete CI/CD flow using Jenkins on AWS EC2:
1) Dockerize a Node.js app
2) Push the image to Docker Hub
3) Build and deploy via Jenkins using a controller and a build agent (SSH)

The repository already includes:
- `dockerfile` for containerization
- `Jenkinsfile` for the CI/CD pipeline
- `k8s/` manifests for deployment

---

## Architecture (High-Level)
- **EC2 #1 (Controller):** Jenkins UI, credentials, pipeline configuration
- **EC2 #2 (Agent):** Executes the pipeline stages (build, push, kubectl)
- **Docker Hub:** Stores built images
- **Kubernetes (Kind):** Runs on the agent for deployment

---

## Repository Structure
- `app.js` – Node/Express app
- `dockerfile` – Docker build instructions
- `Jenkinsfile` – Pipeline definition
- `k8s/deployment.yaml` – Kubernetes Deployment
- `k8s/service.yaml` – Kubernetes Service

---

## Prerequisites
**Accounts & Tools**
- AWS account with ability to create EC2 instances
- Docker Hub account (create a repo like `username/node-k8s-app`)
- Git installed locally

**EC2 Requirements (recommended)**
- Ubuntu 22.04 LTS
- t2.medium or better
- Security Group rules:
  - TCP 22 (SSH) – from your IP
  - TCP 8080 (Jenkins UI) – from your IP
  - TCP 30000–32767 (NodePort access) – from your IP (optional)

---

## Step 1: Clone the Repo
```bash
git clone https://github.com/harishkumar04/Jenkins-pipeline.git
cd Jenkins-pipeline
```

---

## Step 2: Dockerize the App (Local Validation)
The `dockerfile` is already included. Build and test locally:

```bash
docker build -t <dockerhub-username>/node-k8s-app:local .
docker run -p 3000:3000 <dockerhub-username>/node-k8s-app:local
```

Test:
```bash
curl http://localhost:3000/
curl http://localhost:3000/health
```

---

## Step 3: Push the Image to Docker Hub
1) Create a Docker Hub repo (e.g., `node-k8s-app`)
2) Login, tag, and push:

```bash
docker login
docker tag <dockerhub-username>/node-k8s-app:local <dockerhub-username>/node-k8s-app:v1
docker push <dockerhub-username>/node-k8s-app:v1
```

---

## Step 4: Jenkins Controller Setup (EC2 #1)

### 4.1 Install Java 21
```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
java -version
```

### 4.2 Install Jenkins
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /etc/apt/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
```

### 4.3 Unlock Jenkins & Install Plugins
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Open `http://<controller-public-ip>:8080` → paste the password → install **Recommended Plugins**.

### 4.4 Install Docker on Controller (optional but useful)
```bash
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 4.5 Add Docker Hub Credentials
Jenkins UI → **Manage Jenkins** → **Credentials** → **(global)** → **Add Credentials**
- **Kind:** Username/Password  
- **ID:** `dockerhub-creds`  
- **Username/Password:** your Docker Hub login

---

## Step 5: Jenkins Agent Setup with SSH (EC2 #2)

### 5.1 Install Java 21 + Required Tools
```bash
sudo apt update
sudo apt install -y openjdk-21-jdk git docker.io
sudo usermod -aG docker ubuntu
```

### 5.2 Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### 5.3 Install Kind (Kubernetes in Docker)
```bash
curl -Lo kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/
kind version
```

### 5.4 Create a Kind Cluster on Agent
```bash
kind create cluster --name jenkins-kind
kubectl get nodes
```

---

## Step 6: Configure Jenkins Agent (SSH)

### 6.1 Generate SSH Key (on Controller)
```bash
ssh-keygen -t ed25519 -C "jenkins-agent" -f /home/ubuntu/.ssh/jenkins-agent
```

### 6.2 Copy Public Key to Agent
```bash
ssh-copy-id -i /home/ubuntu/.ssh/jenkins-agent.pub ubuntu@<agent-public-ip>
```

### 6.3 Add Agent in Jenkins
Jenkins UI → **Manage Jenkins** → **Nodes and Clouds** → **New Node**
- **Node name:** `docker-agent`
- **Type:** Permanent Agent
- **Remote root directory:** `/home/ubuntu/jenkins`
- **Labels:** `docker`
- **Launch method:** Launch agents via SSH
  - **Host:** agent public IP  
  - **Credentials:** add the private key from `/home/ubuntu/.ssh/jenkins-agent`

Click **Save** → ensure agent status becomes **online**.

---

## Step 7: Create the Jenkins Pipeline Job
Jenkins UI → **New Item** → **Pipeline**
- **Pipeline from SCM**
- **SCM:** Git
- **Repository URL:** `https://github.com/harishkumar04/Jenkins-pipeline.git`
- Jenkinsfile is auto-detected

---

## Step 8: Run the Pipeline
Click **Build Now**. The pipeline will:
1. Clone repo  
2. Build Docker image  
3. Push to Docker Hub  
4. Deploy to Kind (update image in `k8s/deployment.yaml`)  


---

## Verify Deployment
On agent:
```bash
kubectl get pods
kubectl get svc
```

Access using NodePort:
```bash
curl http://<agent-public-ip>:<nodeport>
```

---

## Notes for Interviews
- Jenkins controller orchestrates, agent executes builds
- Docker Hub credentials stored securely in Jenkins
- Kubernetes manifests updated dynamically in pipeline
- Java 21 ensures compatibility with current Jenkins LTS
- SSH-based agent configuration is production-standard

---

## Common Troubleshooting
- **Agent offline:** check SSH connectivity and security group rules
- **Docker permission denied:** ensure user is in `docker` group and re-login
- **kubectl fails:** verify Kind cluster exists on agent
- **Pipeline fails at push:** confirm `dockerhub-creds` ID

