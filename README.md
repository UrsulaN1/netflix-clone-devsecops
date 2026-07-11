
# DevSecOps Project - Deploy Netflix Clone Application on AWS using Jenkins

## ⚠️ Important Project Disclaimers

> [!IMPORTANT]
> Please read the following notices carefully before deploying, sharing, or exploring this repository.

---

> **Educational use only.** This is an unofficial streaming UI created to demonstrate DevSecOps workflows. It is not affiliated with or endorsed by Netflix. It uses TMDB metadata when you provide a TMDB v3 API key and is not endorsed or certified by TMDB.

### 🛡️ Production vs. Lab Environment Guidance

This project is configured as a **practical lab guide**. Running all tools on a single host is intended for isolated testing and learning.

For **production workloads**, you must adhere to the following architecture hardening standards:

* **Decoupled Architecture:** Place Jenkins, SonarQube, monitoring suites (Prometheus/Grafana), and the core application on separate dedicated hosts or managed services.
* **In-Transit Security:** Enforce HTTPS/TLS across all public and internal service endpoints.
* **Network Segmentation:** Heavily restrict network access using strict firewall rules, security groups, and private subnets.
* **Data Protection:** Encrypt all persistent storage volumes at rest.
* **Supply Chain Security:** Pin all software versions, package dependencies, and base container image tags to immutable versions.

## I. OVERVIEW

This project demonstrates an end-to-end DevSecOps workflow for deploying a React-based Netflix clone on AWS.

The workflow uses Jenkins for continuous integration and deployment, SonarQube for static code analysis, OWASP Dependency-Check and Trivy for vulnerability scanning, Docker for packaging and deployment, and Prometheus with Grafana for monitoring.

The guide also includes an optional Kubernetes and Argo CD phase for teams that want to move from a single EC2 deployment to GitOps-based delivery on Amazon EKS.

![Logo](./public/assets/DevSecOps.png)
[![Logo](./public/assets/netflix-logo.png)](http://netflix-clone-with-tmdb-using-react-mui.vercel.app/)
![Logo](./public/assets/home-page.png)
*Home Page*

## II. ARCHITECTURE & TOOLCHAIN

| Area | Tool or Service | Purpose |
| :--- | :--- | :--- |
| **Cloud platform** | AWS EC2 / Amazon EKS | Compute and optional Kubernetes orchestration |
| **Source control** | GitHub | Application and pipeline source |
| **CI/CD** | Jenkins | Automated build, scan, push, and deployment |
| **Code quality** | SonarQube | Static analysis and Quality Gate enforcement |
| **Dependency security** | OWASP Dependency-Check | Known-vulnerability analysis for dependencies |
| **Container security** | Trivy | Filesystem, secret, configuration, and image scanning |
| **Packaging** | Docker | Reproducible application image |
| **Registry** | Docker Hub | Container image storage |
| **Monitoring** | Prometheus, Node Exporter, Grafana | Metrics collection and visualization |
| **GitOps** | Argo CD | Optional Kubernetes deployment automation |

## III.📋 PREREQUISITES

Before starting, ensure you have prepared the following:

* **☁️ AWS Account:** Active account with administrative permissions to provision EC2 instances, Security Groups, IAM Roles, and optionally EKS clusters.
* **🐙 GitHub Repository:** A configured repository containing the application source code and this `README.md`.
* **🎬 TMDB Account:** A registered The Movie Database (TMDB) account with an active API Key generated for application authentication.
* **🐳 Docker Hub Account:** A Docker Hub registry account for hosting and pulling your container images.
* **🌐 Custom Domain (Optional):** A valid domain name and TLS/SSL certificate configured if you plan to deploy to a production environment.
* **🧠 Core Knowledge:** A foundational familiarity with Linux administration, Docker containerization, Git version control, Jenkins pipelines, and AWS networking concepts.

## IV 💻 Recommended EC2 Capacity

If you plan to run all lab components together on a single host, provision an EC2 instance that meets or exceeds the following baseline specifications:

| Resource | Minimum Requirement | Recommended Specification |
| :--- | :--- | :--- |
| **OS Platform** | Ubuntu Server 24.04 LTS | Ubuntu Server 24.04 LTS |
| **Compute** | 4 vCPUs | 4 vCPUs or higher |
| **Memory (RAM)** | 8 GiB | 16 GiB |

> ⚠️ **Important Deployment Note:** Running tools like Jenkins, SonarQube, Prometheus, Grafana, and Docker containers simultaneously is resource-intensive. Staying at or above the **16 GiB RAM** recommendation will prevent out-of-memory errors and pipeline crashes during high-load compilation or scanning stages.

## 2.2.2. Security Group Guidance

Restrict each port to the smallest possible source range or security group.

| **Port**   | **Service**         | **Recommended source**                       |
|------------|---------------------|----------------------------------------------|
| 22         | SSH                 | Your administrator IP only                   |
| 8080       | Jenkins             | Your administrator IP, VPN, or reverse proxy |
| 9000       | SonarQube           | Jenkins security group and administrator IP  |
| 8081       | Netflix application | Intended users or load balancer              |
| 9090       | Prometheus          | Monitoring administrators only               |
| 3000       | Grafana             | Monitoring administrators only               |
| 9100       | Node Exporter       | Prometheus host or security group only       |
| 30000-     | Argo CD             | Argo CD                                      |

Do not expose administrative services to `0.0.0.0/0` unless the environment is temporary and isolated.

<!-- markdownlint-disable MD033 -->
<h1 align="center">🌟 PHASE 1: Initial Setup and Deployment</h1>

## 💡 <u>STEP 1.1: Launch EC2 (Ubuntu Server 24.04 LTS):</u>

* Provision an EC2 instance on AWS with Ubuntu 24.04 with at least a 4GiB Memory and 20 GiB gp3 root volume
* Connect to the instance.

## 💡 <u>STEP 1.2: Clone the Code: </u>

```bash
sudo apt-get update
git clone https://github.com/UrsulaN1/netflix-clone-DevSecOps.git
cd netflix-clone-DevSecOps
```

## 💡 <u>STEP 1.3: Install Docker and Run the App Using a Container:</u>

* Set up Docker on the EC2 instance and add user to docker group to use sudo privileges:

```bash
sudo apt-get update
sudo apt-get install docker.io -y   # this command create the docker group automatically. Otherwise, create group with "sudo groupadd docker"
sudo usermod -aG docker $USER
newgrp docker
```

## 💡 <u>STEP 1.4: Create Docker Image Using Your Movie Database API Key:</u>

* Open a web browser and navigate to TMDB (The Movie Database) website on [https://themoviedb.org]
* Click on "Login" and create an account.
* Once logged in, go to your profile and select "Settings."
* Click on "API" from the left-side panel.
* Create a new API key by clicking "Create" and accepting the terms and conditions.
* Provide the required basic details and click "Submit."
* You will receive your TMDB API key.
* Build and run your application using Docker containers:

```bash
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
```

<h1 align="center">🌟 PHASE 2: JENKINS & SECURITY SETUP</h1>

## 💡 <u>STEP 2.1: Install Jenkins for Automation:</u>

* Install Jenkins on the EC2 instance to automate deployment:

```bash
# java Installation
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java -version
    
#Jenkins Installation
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins --no-pager
```

* Access Jenkins in a web browser using the public IP of your EC2 instance.

```http://<publicIp>:8080```

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## 💡 <u>STEP 2.2: Install Necessary Plugins in Jenkins:</u>

Navigate to **Manage Jenkins** → **Plugins** → **Available Plugins** and install:

* Eclipse Temurin Installer
* SonarQube Scanner
* NodeJs
* OWASP Dependency-Check
* Docker Pipeline
* Kubernetes CLI
* Email Extension
* Workspace Cleanup
* Pipeline stage view
* Prometheus metrics  (restart on success)

Ensure the following system configurations and tool mappings are completed:

* JDK tool named `jdk17`
* NodeJS tool named `node24`
* SonarScanner tool named `sonar-scanner`
* SonarQube server named `sonar-server`
* Secret-text credential `tmdb-api-key`
* Docker registry credential `docker`
* Kubernetes credential `k8s`
* OWASP Dependency-Check installation `DP-Check`
* Trivy installed on the Jenkins agent filesystem

## 💡 <u>STEP 2.3: Configure Java and Nodejs in Global Tool Configuration:</u>

Navigate to **Manage Jenkins** → **Tools**

* Add and configure **jdk17** and **nodejs16**
* Click on Apply and Save

## 💡 <u>STEP 2.4: Install SonarQube and Trivy Platforms:</u>

* Run SonarQube and Trivy containers on the EC2 instance to scan for vulnerabilities.

### SonarQube

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access URL: ```http://<publicIP>:9000``` (Default credentials: admin / admin)

### Trivy

```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy      
```

## 💡 <u>STEP 2.6: Generate the SonarQube Token:</u>

**Inside the SonarQube interface:** Navigate to **Profile icon** → **My Account** → **Security**

Under Generate Tokens:

Enter a descriptive name, such as ```***jenkins-token***```.
Select a token type and choose an expiration date.
Click Generate and copy the token value.

## 💡 <u>STEP 2.6: Add the SonarQube token in Jenkins:</u>

Navigate to **Manage Jenkins** → **Credentials** → **Add credentials**

```groovy
Kind:        Secret text
Scope:       Global
Secret:      Paste your SonarQube token
ID:          Sonar-token
Description: SonarQube authentication token
```

* Click on Apply and Save

## 💡 <u>STEP 2.7: Configure the SonarQube webhook:</u>

In SonarQube, navigate to: **Administration** → **Configuration** → **Webhooks** → **Create**

Enter configuration parameters:

**Name**: Jenkins
**URL**: http://JENKINS_PRIVATE_IP:8080/sonarqube-webhook/

## 💡 <u>STEP 2.8: Configure the SonarQube server systems in Jenkins:</u>

Navigate to **Manage Jenkins** → **System** → **SonarQube servers**
**Name**: sonar-server
**Server URL**: [http://sonarqube-PUBLIC-IP]:9000
**Server authentication token**: Sonar-token

Navigate to **Manage Jenkins** → **Tools** → **SonarQube Scanner Installations**

**Name**: sonar-scanner

## 💡 <u>STEP 2.9: Create Netflix Project in SonarQube UI:</u>

Select **Projects** --> **Manual** --> **Create Project:**

* **Project Name**: "Netflix"
* **Branch name**: main
* Click **Create**, select **With Jenkins** --> **GitHub** and proceed to configure the analysis properties

## 💡 <u>STEP 2.10: Configure the OWASP Dependency-Check Tool:</u>

Navigate to **Manage Jenkins** → **Plugins** → **Available Plugins** and install:

* OWASP Dependency-Check
* Docker
* Docker Pipeline

## 💡 <u>STEP 2.8: Configure the OWASP Tool in Jenkins:</u>

Navigate to **Manage Jenkins** → **Tools**

Locate the section for "OWASP Dependency-Check."
**Name**: DP-Check
Save settings.

<h1 align="center">🌟 PHASE 3: CI/CD PIPELINE SETUP</h1>

## 💡 <u>STEP 3.1: Configure CI/CD Pipeline Execution:</u>

* From the Jenkins dashboard, click **New Item** and select **Pipeline**
* Insert the initial pipeline specification code framework below into your pipeline script configuration:

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/UrsulaN1/netflix-clone-DevSecOps.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
}
```

* Save and execute via `**Build Now**` to perform initial testing

<h1 align="center">🌟 PHASE 4: CONTAINER BUILD, SCANNING, AND DEPLOYMENT</h1>

## 💡 <u>STEP 4.1: Inject DockerHub Credentials into Jenkins:</u>

### 1. Generate a Docker Hub access token

Inside Docker Hub, navigate to **Profile** → **Account settings** → **Personal access tokens**  → **Generate new token**

* **Description**: Jenkins
* **Expiration**: Choose an appropriate date
* **Permissions**: Read & Write

Copy the token immediately.

### 2. Add the credential to Jenkins

Navigate to **Manage Jenkins** → **Credentials** → **Add credentials**

* **Kind**:        Username with password
* **Scope**:       Global
* **Username**:    Your Docker Hub username (treat as secret)
* **Password**:    Paste your Docker Hub access token
* **ID**:          dockerhub
* **Description**: dockerhub

```ℹ️ Note: Use the **Username** with **password** option rather than **Secret text** because the Jenkins Docker Pipeline orchestration syntax explicitly expects a standard pair interface.**```
Create.

## 💡 <u>STEP 4.2: Map Docker Installation Tool</u>

Navigate to **Jenkins UI** → **Manage Jenkins** → **Tools** → **Docker Installations**

* **Name**: dockerhub
* Install automatically from docker.com

Save

## 💡 <u>STEP 4.3: Create an Account and Generate a TMDB API Key from [https://www.themoviedb.org/]</u>

Profile → API subscription → Generate API Key

### Add API key to Jenkins Credentials

Navigate to **Jenkins UI** → **Manage Jenkins** → **Credentials**

* Click on the (`global`) domains link under the Stores scoped to Jenkins section.

* Click `Add Credentials` on the top right.

-Configure the credentials with the following settings:

* `Kind`: Select Secret text from the dropdown.

* `Scope`: Keep it as Global (or restrict it to a specific folder if needed).

* `Secret`: Paste your TMDB API Key here.

* `ID`: Enter a unique identifier, like **tmdb-api-key**. (You will use this ID in your pipeline code).

* `Description`: Add a helpful note, like TMDB API Key for movie app.

* Click `Create`.

## 💡 <u>STEP 4.4: Grant Pipeline Local Host Permissions</u>

Execute these mapping commands on the local underlying host instance filesystem to enable proper execution without elevation issues:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo systemctl status jenkins --no-pager
```

## 💡 <u>STEP 4.5: Execute Full Automation Build Pipeline</u>

* Configure your core workflow orchestration with this comprehensive multi-scanner deployment script block:

```groovy

pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/UrsulaN1/netflix-clone-DevSecOps.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {   
                        withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_KEY')]) {
                    
                            sh "docker build --build-arg TMDB_V3_API_KEY=${TMDB_KEY} -t netflix ."
                            sh "docker tag netflix ursulan1/netflix:latest"
                            sh "docker push ursulan1/netflix:latest"   
                        }
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image ursulan1/netflix:latest > trivyimage.txt" 
            }
        }
        stage("Deploy Container") {
            steps {
                sh "docker stop netflix || true"
                sh "docker rm netflix || true"
                sh "docker run -d --name netflix -p 8081:80 ursulan1/netflix:latest"
            }
        }
    }
}
```

---

<h1 align="center">🌟 PHASE 5: PROMETHEUS MONITORING</h1>

## 💡 <u>STEP 5.1 Deploy Prometheus Engine:</u>

   Set up Prometheus and Grafana to monitor your application.

* Create a dedicated service execution user context:

```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Pull and extract runtime binaries:
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/

# Restructure folder tree locations:
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

# Construct the tracking lifecycle wrapper service unit definition:
sudo nano /etc/systemd/system/prometheus.service
```

* Populate the block with the following operational specification details:</u>

```plaintext
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
 --config.file=/etc/prometheus/prometheus.yml \
 --storage.tsdb.path=/data \
 --web.console.templates=/etc/prometheus/consoles \
 --web.console.libraries=/etc/prometheus/console_libraries \
 --web.listen-address=0.0.0.0:9090 \
 --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

* Initialize structural service runtime systems:

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus --no-pager
```

## 💡 <u>STEP 5.2 Install Node Exporter Metrics Daemon:</u>

* Establish independent daemon tracking accounts:

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Pull artifact binary components:
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

* Construct service mapping units:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```plaintext
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

* Activate collector routines:

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter --no-pager
```

## 💡 <u>STEP 5.3 Update Scraping Targets:</u>

Inject scraper tasks into the central structural definition configuration at `sudo nano /etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<your-jenkins-ip>:8080']
```

   Make sure to replace `<your-jenkins-ip>` and `<your-jenkins-port>` with the appropriate values for your Jenkins setup.

   Check the validity of the configuration file:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

`http://<your-prometheus-ip>:9090/targets`

### Generate an API Token in Jenkins

* Create a dedicated user account in Jenkins (e.g., `prometheus`) or use your own account.

* Click on the `username` in the top-right corner of Jenkins and click `Configure`.

* Find the `API Token` section, click `Add new Token`, give it a name like **prometheus-scrape**, and click `Generate`.

* Copy the secret token immediately—you won't be able to see it again!

### Update your prometheus.yml configuration

`sudo nano /etc/prometheus/prometheus.yml`; add a basic_auth section to your Jenkins job using that username and the generated token:

```YAML
- job_name: 'jenkins'
    metrics_path: '/prometheus'
    basic_auth:
      username: 'your-jenkins-username'
      password: '11adxxxxxxxxxxxxxxxxxxxx' # <--- Paste your copied API token here
    static_configs:
      - targets: ['<Your-Prometheus-server-IP>:8080']
```

## 💡 <u>STEP 5.4: Install Grafana Analytics Engine:</u>

* Install core runtime package requirements:*

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common wget
```

* Map external security validation keys and repositories:

```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

* Sync dependencies and trigger runtime service deployments:

```bash
sudo apt-get update
sudo apt-get -y install grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server --no-pager
```

## 💡 <u>STEP 5.5: Configure Visualizations in Grafana:</u>

* Navigate to dashboard interface endpoint: `http://<your-server-ip>:3000` (Use admin / admin for initial entry).

**Map Data Source**:

Navigate to **Connections** → **Data Sources** → **Add Data Source**

* Select **Prometheus**, specify internal service destination endpoint URL `http://localhost:9090`, then select **Save & Test**.

**Import Dashboards**:

* Select **Create (+)** → **Import Dashboard**
* Input template identifier code reference `1860`
* Click *Load*
* Click **Import**.

HURRAY! You've successfully installed and set up Grafana to work with Prometheus for monitoring and visualization.

<h1 align="center">🌟 PHASE 6: EMAIL NOTIFICATIONS</h1>

## 💡 <u>STEP 6.1: Configure Alerting Mechanisms:</u>

### 1. Configure email notification servers or webhooks within your global setup parameters

* Navigate to **Manage Jenkins** → **System** → **Extended E-mail Notification**

-`SMTP Server`: Enter your provider's SMTP address (e.g., smtp.gmail.com or smtp.office365.com).

-`SMTP Port`:

* Use `465` if you are using `SSL`.

* Use `587` if you are using `TLS`.

### 2. Set the Default Content Type & Recipients

* Scroll slightly down within that same section to set up your global defaults:

* `Default Content Type`: Change this to HTML (text/html) if you want clean, formatted alert emails.

* `Default Recipients`: Add the email addresses that should receive these build alerts by default (separate multiple emails with a comma).

* `Default Subject`: You can leave this as the default token, which usually looks like: $PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!

<h1 align="center">🌟 PHASE 7: KUBERNETES & ARGO CD</h1>

## 💡 <u>STEP 7.1: Install Kubernetes Client (kubectl) and Helm Package Manager:**</u>

* To manage your cluster and deploy containerized ecosystem tools, install kubectl and Helm directly on your Jenkins/EC2 controller instance.

### 1. Install kubectl CLI

```bash
sudo snap install kubectl --classic
# Verify connection capabilities
kubectl version --client
```

### 2. Install Helm 3

```bash
# Download and execute the official Helm installation script
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
# Verify installation
helm version
```

## 💡 <u>STEP 7.2 Install AWS CLI and Configure EC2 Instance Profile</u>

* Before provisioning cloud infrastructure via the terminal, your EC2 host requires programmatic authority to manage AWS resources.

### 1. Install AWS CLI v2

```bash
# 1. Download the official AWS CLI v2 zip package
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 2. Unzip it and run installer script (if you don't have unzip, run 'sudo apt install unzip' first)
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 2. Create and Attach the IAM Identity Role

* Go to the **AWS Console** $\rightarrow$ **IAM** $\rightarrow$ **Roles** $\rightarrow$  **Create role**.
* Select AWS service and choose **EC2**.
* Attach the necessary policies for `full administration` capacity (e.g., `AdministratorAccess` or dedicated policy paths for **CloudFormation**, **EC2**, **VPC**, and **EKS management**).
* Name and save the role (e.g., **Jenkins-EKS-Manager-Role**).
* Go to your EC2 Dashboard, select your running instance, click **Actions** $\rightarrow$ **Security** $\rightarrow$ **Modify IAM role**.
* Select your new role and save.

## 💡 <u>STEP 7.3 Install **eksctl** and Provision the Amazon EKS Cluster</u>

* eksctl automates the creation of your Kubernetes control plane, subnets, and node group components via AWS CloudFormation.

### 1. Install eksctl Binary

```bash
# Download and extract the latest eksctl binary
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# Move the binary to your local bin directory
sudo mv /tmp/eksctl /usr/local/bin

# Verify it works
eksctl version
```

### 2. Provision the Managed EKS Cluster

* Execute the cluster template command. Note: The master orchestration provisioning routine typically takes 10 to 15 minutes to complete.

```bash
eksctl create cluster \
  --name <your-cluster-name> \
  --region <your region> \
  --nodegroup-name <node-group-name> \
  --node-type <instance-type-with-atleast-a-4vcpu-4Gi-memory> \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed
```

## 💡 <u>STEP 7.4 Configure Local Cluster Access and EKS IAM Access Entries</u>

* Ensure your EC2 local terminal and your AWS Web Management Console identity both hold absolute cluster administrative rights.

### 1. Update Local Kubeconfig Matrix

```bash
aws eks update-kubeconfig --region <your-region> --name <your-cluster-name>
```

### 2. Register Console Web Access Entries

* Run this snippet directly from your EC2 terminal to authorize your personal AWS browser login identity to view and track cluster components:

```bash
# 1. Create an access entry for your Web Console IAM identity
aws eks create-access-entry \
    --cluster-name netflix-k8s-cluster \
    --principal-arn arn:aws:iam::User-Account_ID:user/YOUR_CONSOLE_USER_NAME \
    --type STANDARD

# 2. Bind the Cluster Admin policy to that identity
aws eks associate-access-policy \
    --cluster-name netflix-k8s-cluster \
    --principal-arn arn:aws:iam::User-Account_ID:user:user/YOUR_CONSOLE_USER_NAME \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
    --access-scope type=cluster
```

## 💡 <u>STEP 7.5 Deploy Node Exporter DaemonSet via Helm</u>

* Deploy Node Exporter across your active Kubernetes worker nodes to collect infrastructure performance metrics.

```bash
# Add the Prometheus community charts repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Segregate your tracking workspace using dedicated namespaces
kubectl create namespace prometheus-node-exporter

# Deploy the DaemonSet charts collection
helm install cluster-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter

# Confirm all pod containers are spinning up successfully
kubectl get pods -n prometheus-node-exporter -o wide
```

## 💡 <u>STEP 7.6: Integrate Kubernetes Targets into Prometheus Scraper Engine</u>

* Connect the new EKS cluster infrastructure metrics streams to your existing standalone Prometheus engine running on your EC2 base host.

### 1. Extract Your Kubernetes Worker Node Public IP

```bash
kubectl get nodes -o wide
```

`(Locate the EXTERNAL-IP string for your running worker node)`

### 2. Inject Scraping Tasks into Prometheus Configuration

Open the system configuration file:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

* Append the following configuration target block to the end of your scrape_configs section:

```YAML
- job_name: 'Netflix-K8s-Cluster'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['<YOUR-K8S-NODE-EXTERNAL-IP>:9100']
```

### 3. Reload the Prometheus Configuration Matrix

```bash
curl -X POST http://localhost:9090/-/reload
```

* Verify target status by navigating to: `http://<YOUR-EC2-PUBLIC-IP>:9090/targets`

## 💡 <u>STEP 7.7 Deploy and Access Argo CD Core Services</u>

* Argo CD handles automated synchronization and deployment using a declarative GitOps architectural framework.

### 1. Deploy Argo CD to the Cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Expose the API Server via NodePort

* Change the service type from ClusterIP to NodePort to grant browser-based management access:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

### 3. Retrieve the Auto-Generated Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### 4. Fetch the NodePort Port Mapping Value

```bash
kubectl get svc argocd-server -n argocd
```

* Identify the mapped port number `(e.g., 30000+)`. **Add port to the node's security group inbound rules**
* Open your browser and navigate to: `https://<YOUR-K8S-NODE-EXTERNAL-IP>:<NODEPORT>` to log in with username admin and the auto-generated password above.

`kubectl get nodes -o wide`

## 💡 <u>STEP 7.8: Connect GitHub Repository and Deploy Application</u>

* Now, map your declarative tracking manifests to the continuous delivery sync engine.

### 1. Register Source Repository inside Argo CD UI

* Navigate to **Settings** $\rightarrow$ **Repositories** $\rightarrow$ **Connect Repo**.

Select connection method **Via HTTPS**, enter repository URL: `[https://github.com/UrsulaN1/netflix-clone-DevSecOps.git]` and **select Connect**.

### 2. Construct the Application Object

* Click **New App** inside the dashboard and configure the following tracking parameters:

**Application Name**: `netflix-clone`

**Project Name**: `default`

**Sync Policy**: `Automatic` (Check options: Prune Resources & Self Heal)

**Repository URL**: `[https://github.com/UrsulaN1/netflix-clone-DevSecOps.git]`

**Revision**: `main`

**Path**: `Kubernetes`/ (Or the subdirectory containing your deployment and service YAML manifests)

**Cluster URL**: `[https://kubernetes.default.svc]`

**Namespace**: `default`

Click **Create** and monitor the visual dependency graph as it synchronizes state.

## 💡 <u>STEP 7.9: Verify and Access the Application Endpoints</u>

* Your deployment manifests route inbound user web traffic through target port 30007.

1. Audit Live Cluster Service Status

```bash
kubectl get service netflix
```

### Your DevSecOps lifecycle transformation pipeline is now complete

## 💡 <u>Delete Active Infrastructure to Avoid Cloud Costs</u>

* Run these commands in order from your EC2 terminal workspace to tear down your laboratory components and drop your billing footprint back to zero.

```bash
# 1. Delete the ArgoCD orchestration tracking wrapper
kubectl delete -n argocd application netflix-clone --cascade

# 2. Uninstall metrics agents and drop dedicated workspace naming scopes
helm uninstall cluster-node-exporter --namespace prometheus-node-exporter
kubectl delete namespace prometheus-node-exporter
kubectl delete namespace argocd

# 3. Terminate LoadBalancer definitions to trigger AWS ELB component teardown
kubectl delete svc --all --all-namespaces

# 4. Tear Down the EKS Control Plane and associated Worker Node fleets
eksctl delete cluster --name <cluster-name>> --region <region>
```

`* If you created the cluster manually via the AWS Management Console:`

-Navigate to **Elastic Kubernetes Service** ➔ **Clusters**.

-Click your **cluster name**, go to the **Compute tab**, select your **Node Groups**, and click **Delete**.

-Once the Node Groups are completely deleted, go back to the cluster page and click **Delete Cluster**.

* Terminate all running/stopped instances
* Clean Up Detached Persistent Volumes

`*  FINAL SANITY CHECK: The AWS Billing Dashboard`

To guarantee everything is completely gone and your spend footprint is dropped to zero:

-Navigate to the **AWS Billing and Cost Management** console.

-Review the Bills section over the **next 24 hours**.

-Ensure no resources under **Elastic Compute Cloud**, **EBS**, or **EKS** are continuing to tick upward.
