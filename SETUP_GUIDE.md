# CI/CD Pipeline Setup Guide

This guide provides step-by-step instructions for configuring your Jenkins pipeline on AWS EC2 to automate fetching code, building Docker images, deploying to Kubernetes, and monitoring.

## 1. GitHub Webhook Setup
To trigger Jenkins builds automatically when you push code to GitHub:

1.  **Get your Jenkins Webhook URL**:
    *   The URL format is: `http://<YOUR_EC2_IP>:8080/github-webhook/`
    *   Example: `http://3.237.47.150:8080/github-webhook/`
2.  **Go to your GitHub Repository**: `https://github.com/popeye605/straming-website`
3.  **Navigate to Settings**: Click on the **Settings** tab at the top.
4.  **Webhooks**: Click on **Webhooks** in the left sidebar.
5.  **Add webhook**:
    *   **Payload URL**: Enter your Jenkins Webhook URL.
    *   **Content type**: Select `application/json`.
    *   **Secret**: (Optional) Leave blank unless configured in Jenkins.
    *   **Which events?**: Select "Just the `push` event."
    *   Ensure **Active** is checked.
6.  **Click "Add webhook"**. GitHub will send a ping to verify.

---

## 2. Jenkins Plugin Requirements
Ensure the following plugins are installed (Manage Jenkins -> Plugins -> Available Plugins):
*   **Git**
*   **GitHub Integration**
*   **Pipeline**
*   **Docker Pipeline**
*   **Kubernetes CLI**
*   **NodeJS** (required for building the app)

---

## 2.5 EC2 Environment Preparation
Run these on your EC2 instance if not already installed:

### Install Docker
```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker jenkins  # Allow Jenkins to run docker
sudo systemctl restart jenkins
```

### Install Kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

---

## 3. Jenkins Credentials Configuration
Navigate to **Manage Jenkins -> Credentials -> System -> Global credentials (unrestricted)** and add the following:

| ID | Kind | Status | Description |
| :--- | :--- | :--- | :--- |
| `docker` | Username with password | ✅ Present | Logged in as `popeye605`. |
| `k8s-creds` | Secret file / Kubeconfig | ❌ Missing | Upload your `.kube/config` file. |
| `TMDB_API_KEY` | Secret text | ❌ Missing | Your The Movie Database API Key. |

> [!TIP]
> Since you have your SSH key in "downloads", you might also need to add a credential of type "SSH Username with private key" if you are using private Git repos, or if Jenkins needs to SSH into worker nodes.

---

## 4. Configuring the Jenkins Pipeline Job
1.  **Create New Item**: Select **Pipeline** and name it (e.g., `Netflix-CI-CD`).
2.  **Build Triggers**: Check **GitHub hook trigger for GITScm polling**.
3.  **Pipeline Definition**:
    *   Select **Pipeline script from SCM**.
    *   **SCM**: Git.
    *   **Repository URL**: `https://github.com/popeye605/straming-website.git`
    *   **Branch Specifier**: `*/main`
    *   **Script Path**: `Jenkinsfile`
4.  **Save**.

---

## 5. Monitoring Setup (Prometheus & Grafana)
To monitor your application once deployed:

### Install Prometheus & Node Exporter on EC2
```bash
# Update and Install
sudo apt update
sudo apt install prometheus prometheus-node-exporter -y

# Check status
sudo systemctl status prometheus
```

### Install Grafana
```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```
*   **Access Grafana**: `http://3.237.47.150:3000/` (Default login: `admin`/`admin`)
*   **Connect Prometheus**: Add Prometheus as a Data Source in Grafana (`http://localhost:9090`).

---

## 6. Accessing the Application
Once the pipeline runs successfully, the application is deployed as a **NodePort** service on Kubernetes.
*   **URL**: `http://<K8S_NODE_IP>:30007` (If using MiniKube or direct EC2 nodes).
*   **Note**: Ensure AWS Security Groups allow inbound traffic on port `30007` (Deployment), `3000` (Grafana), `8080` (Jenkins), and `9090` (Prometheus).
