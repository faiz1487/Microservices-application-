#  Online shopping E-commerce Website and 11 MicroService DevSecops Project with K8s.

## This project Forked from the https://github.com/GoogleCloudPlatform/microservices-demo (Org source)
## https://github.com/faiz1487/Microservices-application-.git

**Online Boutique** is a cloud-first microservices demo application.  The application is a
web-based e-commerce app where users can browse items, add them to the cart, and purchase them.


## Architecture

[![Architecture of
microservices](/docs/img/architecture-diagram.png)](https://youtu.be/KNH_qe1vJAg)


| Service                                              | Language      | Description                                                                                                                       |
| ---------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [frontend](/src/frontend)                           | Go            | Exposes an HTTP server to serve the website. Does not require signup/login and generates session IDs for all users automatically. |
| [productcatalogservice](/src/productcatalogservice) | Go            | Provides the list of products from a JSON file and ability to search products and get individual products.                        |
| [shippingservice](/src/shippingservice)             | Go            | Gives shipping cost estimates based on the shopping cart. Ships items to the given address (mock)                                 |
| [checkoutservice](/src/checkoutservice)             | Go            | Retrieves user cart, prepares order and orchestrates the payment, shipping and the email notification.                            |
| [cartservice](/src/cartservice)                     | C#            | Stores the items in the user's shopping cart in Redis and retrieves it.                                                           |
| [currencyservice](/src/currencyservice)             | Node.js       | Converts one money amount to another currency. Uses real values fetched from European Central Bank. It's the highest QPS service. |
| [paymentservice](/src/paymentservice)               | Node.js       | Charges the given credit card info (mock) with the given amount and returns a transaction ID.                                     |
| [emailservice](/src/emailservice)                   | Python        | Sends users an order confirmation email (mock).                                                                                   |
| [recommendationservice](/src/recommendationservice) | Python        | Recommends other products based on what's given in the cart.                                                                      |
| [loadgenerator](/src/loadgenerator)                 | Python/Locust | Continuously sends requests imitating realistic user shopping flows to the frontend.                                              |
| [adservice](/src/adservice)                         | Java          | Provides text ads based on given context words.                                                                                   |

## Screenshots

| Home Page                                                                                                         | Checkout Screen                                                                                                    |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [![Screenshot of store homepage](/docs/img/online-boutique-frontend-1.png)](/docs/img/online-boutique-frontend-1.png) | [![Screenshot of checkout screen](/docs/img/online-boutique-frontend-2.png)](/docs/img/online-boutique-frontend-2.png) |


## Table of Contents

- [Prerequisites](#prerequisites)
- [System Update & Common Packages](#system-update--common-packages)
- [Java](#java)
- [Jenkins](#jenkins)
- [Docker](#docker)
- [Trivy](#trivy-vulnerability-scanner)
- [Jenkins Plugins to Install](#jenkins-plugins-to-install)
- [Jenkins Credentials to Store](#jenkins-credentials-to-store)
- [Jenkins Tools Configuration](#jenkins-tools-configuration)
- [Jenkins System Configuration](#jenkins-system-configuration)
- [GKE Kubernetes Setup Guide](#GKE-kubernetes-setup-guide)
- [Monitor Kubernetes with Prometheus](#monitor-kubernetes-with-prometheus)
- [Notes and Recommendations](#notes-and-recommendations)

---

## Prerequisites

- Instance Type :- c5.xlarge
This guide assumes an Ubuntu/Debian-like environment and sudo privileges.

---

## Ports to Enable in Security Group

| Service         | Port  |
|-----------------|-------|
| HTTP            | 80    |
| HTTPS           | 443   |
| SSH             | 22    |
| Jenkins         | 8080  |
| SonarQube       | 9000  |

## System Update & Common Packages

```bash
sudo apt update
sudo apt upgrade -y

# Common tools
sudo apt install -y bash-completion wget git zip unzip curl jq net-tools build-essential ca-certificates apt-transport-https gnupg fontconfig
```
Reload bash completion if needed:
```bash
source /etc/bash_completion
```

**Install latest Git:**
```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git -y
```

---

## Java

Install OpenJDK (choose 17 or 21 depending on your needs):

```bash
# OpenJDK 17
sudo apt install -y openjdk-17-jdk

# OR OpenJDK 21
sudo apt install -y openjdk-21-jdk
```
Verify:
```bash
java --version
```

---

## Jenkins

Official docs: https://www.jenkins.io/doc/book/installing/linux/

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
Initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then open: http://your-server-ip:8080

**Note:** Jenkins requires a compatible Java runtime. Check the Jenkins documentation for supported Java versions.

---

## Docker

Official docs: https://docs.docker.com/engine/install/ubuntu/

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group (log out / in or newgrp to apply)
sudo usermod -aG docker $USER
newgrp docker
docker ps
```
If Jenkins needs Docker access:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
Check Docker status:
```bash
sudo systemctl status docker
```

---

## Trivy (Vulnerability Scanner)

Docs: https://trivy.dev/v0.65/getting-started/installation/

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy


trivy --version
```

## Python Package Installation in the Amazon Ubuntu Ami Image [ Preinstalled 3.12 ]

```bash
# 1. Update package list
sudo apt update

# 2. Install required dependencies for adding a new Python version
sudo apt install -y software-properties-common

# 3. Add the deadsnakes PPA (Personal Package Archive) to get newer Python versions
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# 4. Install the specific version of Python you want
sudo apt install -y python3.10 python3.10-venv python3.10-distutils python3.10-dev

curl -sS https://bootstrap.pypa.io/get-pip.py | sudo python3.10


ls /usr/bin/python3*

sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.12 2
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1


sudo update-alternatives --config python3

```
## Choose python3.10 for Jenkins pipeline.

## If your using the Plan VM
```bash
sudo apt-get update
sudo apt install -y python3.10 python3.10-venv python3.10-distutils python3.10-dev
```
---

## Jenkins Plugins to Install

- Eclipse Temurin installer Plugin
- NodeJS
- Email Extension Plugin
- OWASP Dependency-Check Plugin
- Pipeline: Stage View Plugin
- SonarQube Scanner for Jenkins
- Go
- .NET SDK Support
- Python Pyenv



---
## SonarQube Docker Container Run for Analysis

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:25.9.0.112764-community
```

---

## Jenkins Credentials to Store

| Purpose       | ID            | Type          | Notes                               |
|---------------|---------------|---------------|-------------------------------------|
| Email         | mail-cred     | Username/app password |                                  |
| SonarQube     | sonar-token   | Secret text   | From SonarQube application         |
| Docker Hub    | docker-cred   | Secret text   | From your Docker Hub profile       |

Webhook example:  
`http://<jenkins-ip>:8080/sonarqube-webhook/`

---

## Jenkins Tools Configuration

- JDK [jdk17 , jdk21 ]
- SonarQube Scanner installations [sonar-scanner]
- Node [ node16 , node20 ]
- Dependency-Check installations [dp-check]
- Go [ go1.25 ]
- .NET SDK installations [ dotnet9 ]
  | .NET 9.0 - Status Unknown (end of support: 2026-11-10) |
  |9.0.9, released 2025-09-09 |
  |9.0.305|
  |linux-x64 (Linux - x64)|
---

## Jenkins System Configuration

**SonarQube servers:**   
- Name: sonar-server  
- URL: http://sonar-ip-address:9000  
- Credentials: Add from Jenkins credentials

**Extended E-mail Notification:**
- SMTP server: smtp.gmail.com
- SMTP Port: 465
- Use SSL
- Default user e-mail suffix: @gmail.com

**E-mail Notification:**
- SMTP server: smtp.gmail.com
- Default user e-mail suffix: @gmail.com
- Use SMTP Authentication: Yes
- User Name: example@gmail.com
- Password: Use credentials
- Use TLS: Yes
- SMTP Port: 587
- Reply-To Address: example@gmail.com

---
# Now See the configuration pipeline of the Jenkins
---
# 🚀 Develop & Deploy Online Boutique on GCP GKE (UI + CLI)

This guide explains **how to develop and deploy the Online Boutique (11 microservices)** on **Google Kubernetes Engine (GKE)** using **GCP Console (UI)** and **CLI commands**.

---

## 🔹 Phase 1: GCP Console (UI) – Initial Setup

### 1️⃣ Create / Select GCP Project

1. Go to **GCP Console → Project Selector**
2. Click **New Project**
3. Project name: `gke-microservices-demo`
4. Click **Create**

---

### 2️⃣ Enable Required APIs

GCP Console → **APIs & Services → Enable APIs**

Enable:
- Kubernetes Engine API
- Artifact Registry API
- Compute Engine API
- Cloud Build API

---

### 3️⃣ Create GKE Cluster (UI Method)

GCP Console → **Kubernetes Engine → Clusters → Create**

Choose:
- Mode: **Standard**
- Cluster name: `online-boutique-gke`
- Region: `asia-south1`
- Zonal: `asia-south1-a`

**Node Pool Configuration**
- Machine type: `e2-standard-4`
- Nodes: `3`
- Image type: Container-Optimized OS

Click **Create**

---

### 4️⃣ Configure kubectl Access

GCP Console → **Clusters → Connect → Run in Cloud Shell**

```bash
gcloud container clusters get-credentials online-boutique-gke \
  --zone asia-south1-a
```

Verify:
```bash
kubectl get nodes
```

---

## 🔹 Phase 2: Artifact Registry (UI + CLI)

### 5️⃣ Create Docker Repository (UI)

GCP Console → **Artifact Registry → Create Repository**

- Name: `microservices-repo`
- Format: Docker
- Location: asia-south1

---

### 6️⃣ Authenticate Docker

```bash
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

---

## 🔹 Phase 3: Application Build & Push

### 7️⃣ Clone the Project

```bash
git clone https://github.com/faiz1487/Microservices-application-.git
cd microservices-demo
```

---

### 8️⃣ Build & Push Docker Images (Example: frontend)

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=asia-south1

cd src/frontend

docker build -t $REGION-docker.pkg.dev/$PROJECT_ID/microservices-repo/frontend:v1 .
docker push $REGION-docker.pkg.dev/$PROJECT_ID/microservices-repo/frontend:v1
```

Repeat for all services or use CI/CD.

---

## 🔹 Phase 4: Kubernetes Deployment

### 9️⃣ Create Namespace

```bash
kubectl create namespace online-boutique
```

---

### 🔟 Update Image Paths in YAML

Edit Kubernetes manifests:

```yaml
image: asia-south1-docker.pkg.dev/PROJECT_ID/microservices-repo/frontend:v1
```

---

### 1️⃣1️⃣ Deploy All Services

```bash
kubectl apply -f ./release/kubernetes-manifests.yaml -n online-boutique
```

Check status:
```bash
kubectl get pods -n online-boutique
```

---

## 🔹 Phase 5: Expose Application

### 1️⃣2️⃣ Create LoadBalancer Service

```bash
kubectl expose deployment frontend \
  --type=LoadBalancer \
  --name=frontend-external \
  -n online-boutique
```

Get External IP:
```bash
kubectl get svc -n online-boutique
```

Open in browser:
```
http://EXTERNAL-IP
```

---

## 🔹 Phase 6: UI Verification (GCP Console)

GCP Console → **Kubernetes Engine → Workloads**
- Verify all 11 services are **Running**

GCP Console → **Services & Ingress**
- Confirm frontend LoadBalancer

---

## Monitor Kubernetes with Prometheus

**Install Node Exporter using Helm:**

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm search repo prometheus-community
```

```bash
kubectl create namespace prometheus
```


```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

```bash
kubectl get pods -n prometheus
```

```bash
kubectl get svc -n prometheus
```

## Edit Prometheus Service
```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
```

## Edit Grafana Service
```bash
kubectl edit svc stable-grafana -n prometheus
```

```bash
kubectl get svc -n prometheus
```

## Grafana Login Details
|                                               |     |                                                                                                                        
| ---------------------------------------------------- | ------------- | 
| UserName | admin |
| Password  | prom-operator |

|                                               |     |                        
| ---------------------------------------------------- | ------------- | 
| Kubernetes Monitoring Dashboard | 12740 |
| Node Exporter | 1860 |
| Kubernetes / Views / Namespace | 15758 |

```
##  Delete EKS Cluster (Cleanup) finally u done a project 

```bash
eksctl delete cluster --name my-cluster --region ap-south-1
```
## 🔹 Phase 9: Cleanup

```bash
gcloud container clusters delete online-boutique-gke \
  --zone asia-south1-a
---

## ✅ Final Result

✔ 11 Microservices running on GKE  
✔ LoadBalancer exposed frontend  
✔ Logs & metrics visible in GCP UI  
✔ Production-ready DevSecOps project
