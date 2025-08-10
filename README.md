# Ultimate-devops-lifecycle
---
 <img width="951" height="648" alt="Screenshot 2025-07-18 at 1 50 55 PM" src="https://github.com/user-attachments/assets/c8525ee0-0f8c-4ebc-921a-7f10683dbe66" />


### Overview of the Project
This project demonstrates a complete end-to-end corporate-level CI/CD DevOps pipeline for a full-stack application (a board game app built with Java Spring Boot, Thymeleaf, and other technologies). The pipeline automates code compilation, testing, quality checks, vulnerability scanning, artifact management, Docker image building and pushing, Kubernetes deployment, email notifications, and monitoring. The infrastructure is set up on AWS EC2 instances running Ubuntu, with Kubernetes for orchestration.

The project is divided into four main phases, based on the video tutorial and supporting resources. I'll provide very detailed steps for each phase, including prerequisites, tools, commands, configurations, and troubleshooting notes. These steps assume basic familiarity with AWS, Linux, and DevOps tools. All setups are done from scratch, using free-tier eligible resources where possible.

**Prerequisites:**
- An AWS account with access to EC2 (free tier for t2.micro instances).
- A local machine with SSH client (e.g., MobaXterm or terminal).
- GitHub account for private repo.
- Docker Hub account for image repository.
- Gmail account for email notifications (enable less secure apps or use app password).
- Basic tools: Git installed locally.
- Application source code: Clone from https://github.com/jaiswaladi246/Boardgame (a Java Spring Boot app with Maven, Thymeleaf templates, H2 database, JUnit tests, and user roles like "bugs/bunny" for user and "daffy/duck" for manager).

**Tools Used Across the Project:**
- Infrastructure: AWS EC2 (Ubuntu 20.04 LTS), Kubernetes (v1.28.1 with kubeadm), Docker/CRI-O.
- CI/CD: Jenkins.
- Code Quality: SonarQube.
- Artifact Management: Nexus.
- Security Scans: Trivy (for source code and Docker images), Kube-bench (for K8s cluster).
- Containerization: Docker.
- Orchestration: Kubernetes with Calico (networking) and NGINX Ingress.
- Monitoring: Prometheus, Grafana, Node Exporter (system metrics), Blackbox Exporter (website monitoring).
- Other: Maven (build), JUnit (tests), Email (SMTP via Gmail).

**Note:** Run all commands as root or with `sudo`. Costs may incur on AWS; monitor usage. If using CRI-O instead of Docker for K8s, follow the blog's variant.

---

### Phase 1: Setup Infrastructure
This phase sets up the isolated network, Kubernetes cluster, security scans, and VMs/servers for Jenkins, SonarQube, and Nexus. Use AWS EC2 for VMs (e.g., 3 instances: one for K8s master, one for worker, and separate ones for tools if needed; but for simplicity, tools can be on separate VMs).

#### Step 1: Create Network Environment and Launch EC2 Instances (Isolated VPC for Privacy)
1. Log in to AWS Console: Go to https://aws.amazon.com/console/ and sign in.
2. Create a VPC for isolation:
   - Search for "VPC" > Dashboards > Create VPC.
   - Name: "DevOps-VPC".
   - IPv4 CIDR: 10.0.0.0/16.
   - Create subnets (e.g., public: 10.0.1.0/24, private: 10.0.2.0/24).
   - Attach Internet Gateway: Create IGW > Attach to VPC.
   - Route Table: Add route 0.0.0.0/0 to IGW.
   - Security Groups: Create one allowing SSH (22), HTTP (80/8080), HTTPS (443), and K8s ports (6443, 10250, etc.).
3. Launch EC2 Instances (Ubuntu 20.04 LTS, t2.micro, at least 2 vCPU/4GB RAM for K8s):
   - Go to EC2 > Instances > Launch Instances.
   - Name: "K8s-Master", "K8s-Worker", "Jenkins-VM", "Sonar-Nexus-VM".
   - AMI: Ubuntu Server 20.04 LTS (free tier eligible).
   - Instance Type: t2.micro (or larger for production).
   - Key Pair: Create or select one (download .pem file).
   - Network: Select your VPC and public subnet for accessibility.
   - Storage: 8-30 GB root volume.
   - Security Group: Allow SSH from your IP, plus tool-specific ports (e.g., 8080 for Jenkins, 9000 for SonarQube, 8081 for Nexus).
   - Launch 4 instances: 1 master, 1 worker for K8s; separate for Jenkins, SonarQube/Nexus.
   - Connect via SSH: `ssh -i key.pem ubuntu@<public-ip>` (use MobaXterm for ease).
4. Update All Instances:
   - On each: `sudo apt-get update && sudo apt-get upgrade -y`.

#### Step 2: Setup Kubernetes Cluster
Use kubeadm for a multi-node cluster (master + worker). Install on K8s-Master and K8s-Worker instances.

1. Install Dependencies (On All Nodes):
   - `sudo apt-get update -y`
   - `sudo apt-get install -y apt-transport-https ca-certificates curl gnupg`
   - `sudo mkdir -p -m 755 /etc/apt/keyrings`
   - Disable Swap: `sudo swapoff -a` (edit /etc/fstab to comment out swap line).
   - Load Kernel Modules: `cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf\noverlay\nbr_netfilter\nEOF`
   - `sudo modprobe overlay && sudo modprobe br_netfilter`
   - Sysctl Params: `cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf\nnet.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1\nEOF`
   - `sudo sysctl --system`

2. Install Container Runtime (Docker or CRI-O; use CRI-O for compatibility):
   - For CRI-O: `sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg`
   - Add Key: `sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg`
   - Add Repo: `echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list`
   - Update & Install: `sudo apt-get update -y && sudo apt-get install -y cri-o`
   - Start CRI-O: `sudo systemctl daemon-reload && sudo systemctl enable --now crio`
   - (Alternative: Docker - `sudo apt install docker.io -y && sudo chmod 666 /var/run/docker.sock`)

3. Install Kubernetes Components (v1.28.1):
   - Add Key: `curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`
   - Add Repo: `echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/' | sudo tee /etc/apt/sources.list.d/kubernetes.list`
   - Update & Install: `sudo apt update && sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1`
   - Hold Versions: `sudo apt-mark hold kubelet kubeadm kubectl`

4. Initialize Master Node (On K8s-Master):
   - `sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/crio/crio.sock` (use your CIDR; note the join command output for workers).
   - Setup Kubeconfig: `mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config`

5. Join Worker Node (On K8s-Worker):
   - Run the join command from master output, e.g., `sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket unix:///var/run/crio/crio.sock`

6. Install Network Add-on (Calico, On Master):
   - `kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml` (wait for pods to be ready: `kubectl get pods -n calico-system`).

7. Install Ingress Controller (NGINX, On Master):
   - `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml` (for cloud; adjust for bare-metal if needed).
   - Verify: `kubectl get pods -n ingress-nginx`.

#### Step 3: Security Scan of K8s Cluster
1. Install Kube-bench (On Master):
   - Download: `curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.8.0/kube-bench_0.8.0_linux_amd64.tar.gz -o kube-bench.tar.gz`
   - Extract: `tar -xvf kube-bench.tar.gz && sudo mv kube-bench /usr/local/bin/`
2. Run Scan: `kube-bench --benchmark cis-1.24` (CIS benchmark for K8s v1.24+).
3. Review Report: Fix issues like enabling RBAC, securing etcd, etc., as per output.

#### Step 4: Setup VMs for Jenkins, SonarQube, & Nexus
Use separate EC2 instances or install on one for simplicity (but separate for production).

1. Install Java (On All Tool VMs):
   - `sudo apt install openjdk-17-jre-headless -y` (for Jenkins/Sonar/Nexus).

2. Setup Jenkins (On Jenkins-VM):
   - Add Key: `sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key`
   - Add Repo: `echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null`
   - Update & Install: `sudo apt-get update && sudo apt-get install jenkins -y`
   - Start: `sudo systemctl start jenkins && sudo systemctl enable jenkins`
   - Access: http://<jenkins-ip>:8080, unlock with initial password from `/var/lib/jenkins/secrets/initialAdminPassword`.
   - Install Plugins: Suggested + Maven Integration, SonarQube Scanner, Nexus Artifact Uploader, Docker Pipeline, Kubernetes, Email Extension.

3. Setup SonarQube (On Sonar-VM):
   - Download: `wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip`
   - Unzip: `unzip sonarqube-9.9.0.65466.zip && mv sonarqube-9.9.0.65466 /opt/sonarqube`
   - Create User: `sudo useradd -r -s /bin/false sonarqube && sudo chown -R sonarqube:sonarqube /opt/sonarqube`
   - Start: As sonarqube user, `./bin/linux-x86-64/sonar.sh start`
   - Access: http://<sonar-ip>:9000, default admin/admin.
   - Generate Token: In SonarQube UI > My Account > Security > Generate Token.

4. Setup Nexus (On Nexus-VM):
   - Download: `wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz`
   - Extract: `tar -xvf latest-unix.tar.gz && mv nexus-3.* /opt/nexus`
   - Create User: `sudo useradd nexus && sudo chown -R nexus:nexus /opt/nexus`
   - Edit Run As: `sudo vim /opt/nexus/bin/nexus.rc` (add `run_as_user="nexus"`).
   - Start: As nexus, `/opt/nexus/bin/nexus start`
   - Access: http://<nexus-ip>:8081, initial password in `/opt/sonatype-work/nexus3/admin.password`.
   - Create Repo: Maven2 hosted for artifacts.

**Troubleshooting:** Ensure ports are open in security groups. For memory issues, increase instance size.

---

### Phase 2: Git Repo Setup
This phase creates a private GitHub repo and pushes the source code.

#### Step 1: Create Private Git Repo
1. Log in to GitHub: Go to https://github.com.
2. Create Repo: New > Name: "Boardgame" (or your app name) > Private > Create.
3. Generate Personal Access Token (PAT): Profile > Settings > Developer Settings > Personal Access Tokens > Fine-grained > Generate (scopes: repo).

#### Step 2: Push Source Code
1. Clone Locally: `git clone https://github.com/jaiswaladi246/Boardgame.git`
2. Add Remote: `cd Boardgame && git remote add private https://github.com/<your-username>/Boardgame.git`
3. Push: `git push -u private main` (use PAT for auth if needed).
4. Verify: Check GitHub UI for files (e.g., pom.xml for Maven, src/main/java for code, schema.sql for DB, Thymeleaf templates in resources/templates).

**Key Files in Boardgame Repo:**
- `pom.xml`: Maven config with dependencies (Spring Boot, Thymeleaf, H2, JUnit, etc.).
- `application.properties`: Config for server port, DB, etc.
- Kubernetes Manifests: Add deployment.yaml, service.yaml (LoadBalancer type).
- Dockerfile: `FROM openjdk:17\nCOPY target/*.jar app.jar\nENTRYPOINT ["java","-jar","/app.jar"]`

**Troubleshooting:** If push fails, use `git push --set-upstream origin main`. Ensure repo is private.

---

### Phase 3: Configure Jenkins & CI/CD Pipeline
Integrate tools in Jenkins, create the pipeline script (Jenkinsfile).

#### Step 1: Configure Jenkins
1. Install Plugins: In Jenkins UI > Manage Jenkins > Plugins > Available: Search and install SonarQube Scanner, Nexus Artifact Uploader, Pipeline: Maven, Docker, Kubernetes Continuous Deploy, Email Extension, Trivy.
2. Global Tool Config: Manage Jenkins > Tools > Add Maven ("maven3"), JDK ("jdk17"), Docker.
3. Credentials: Manage Credentials > Add GitHub PAT, Docker Hub creds, K8s kubeconfig, Sonar token, Nexus creds.
4. SonarQube Integration: System > Add SonarQube server (URL: http://<sonar-ip>:9000, token).
5. Nexus Integration: Add Nexus repo URL and creds.
6. K8s Integration: Install kubectl on Jenkins VM: `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`
   - Copy kubeconfig from master to Jenkins.
7. Email Config: System > Extended Email > SMTP (smtp.gmail.com, port 465, SSL, your Gmail/app password).

#### Step 2: Create CI/CD Pipeline
1. New Item > Pipeline > Scripted or Declarative.
2. Jenkinsfile (commit to Git repo):
   ```
   pipeline {
       agent any
       tools {
           maven 'maven3'
       }
       environment {
           SONAR_TOKEN = credentials('sonar-token')
           NEXUS_CREDS = credentials('nexus-creds')
           DOCKERHUB_CREDS = credentials('dockerhub-creds')
           BUILD_NUMBER = "${env.BUILD_NUMBER}"
       }
       stages {
           stage('Compile') {
               steps { sh 'mvn compile' }
           }
           stage('Unit Tests') {
               steps { sh 'mvn test' }
           }
           stage('Sonar Analysis') {
               steps {
                   withSonarQubeEnv('SonarQube') {
                       sh 'mvn sonar:sonar -Dsonar.login=${SONAR_TOKEN}'
                   }
               }
           }
           stage('Quality Gate') {
               steps {
                   timeout(time: 5, unit: 'MINUTES') {
                       waitForQualityGate abortPipeline: true
                   }
               }
           }
           stage('Trivy FS Scan') {
               steps {
                   sh 'trivy fs --format table -o trivy-fs-report.html .'
               }
           }
           stage('Build Artifact') {
               steps { sh 'mvn package' }
           }
           stage('Publish to Nexus') {
               steps {
                   nexusArtifactUploader artifacts: [[artifactId: 'boardgame', classifier: '', file: 'target/boardgame-0.0.1-SNAPSHOT.jar', type: 'jar']], credentialsId: 'nexus-creds', groupId: 'com.devopsshack', nexusUrl: '<nexus-ip>:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'maven-releases', version: "${BUILD_NUMBER}"
               }
           }
           stage('Build Docker Image') {
               steps { sh "docker build -t <your-dockerhub>/boardgame:${BUILD_NUMBER} ." }
           }
           stage('Trivy Image Scan') {
               steps { sh "trivy image --format table -o trivy-image-report.html <your-dockerhub>/boardgame:${BUILD_NUMBER}" }
           }
           stage('Push to Docker Hub') {
               steps {
                   withDockerRegistry(credentialsId: 'dockerhub-creds', url: 'https://index.docker.io/v1/') {
                       sh "docker push <your-dockerhub>/boardgame:${BUILD_NUMBER}"
                   }
               }
           }
           stage('Deploy to K8s') {
               steps {
                   sh "sed -i 's|image: .*|image: <your-dockerhub>/boardgame:${BUILD_NUMBER}|g' k8s/deployment.yaml"
                   sh "kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml"
                   sh "kubectl get pods && kubectl get svc"
               }
           }
       }
       post {
           always {
               emailext subject: "Pipeline Status: ${currentBuild.currentResult}", body: "Build ${BUILD_NUMBER} - ${currentBuild.currentResult}", to: 'your@email.com'
           }
       }
   }
   ```
3. Trigger Pipeline: Configure webhook from GitHub or manual build.
4. Verify: Check console output for each stage success.

**Troubleshooting:** If Sonar fails, check token. For K8s deploy, ensure service is LoadBalancer and gets external IP.

---

### Phase 4: Monitoring
Setup on a separate VM or K8s.

#### Step 1: Install Prometheus & Grafana
1. Download Prometheus: `wget https://github.com/prometheus/prometheus/releases/download/v2.47.0/prometheus-2.47.0.linux-amd64.tar.gz`
   - Extract: `tar xvfz prometheus-*.tar.gz && cd prometheus-*`
   - Run: `./prometheus --config.file=prometheus.yml &`
2. Edit prometheus.yml: Add jobs for node_exporter (port 9100), blackbox_exporter (port 9115), targets like localhost:9100.
3. Install Grafana: `sudo apt-get install -y adduser libfontconfig1 musl && wget https://dl.grafana.com/oss/release/grafana_11.1.0_amd64.deb && sudo dpkg -i grafana_11.1.0_amd64.deb`
   - Start: `sudo systemctl start grafana-server`
   - Access: http://<grafana-ip>:3000 (admin/admin).

#### Step 2: Setup Exporters
1. Node Exporter (System Monitoring, on monitored VM e.g., Jenkins):
   - Download: `wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz`
   - Extract & Run: `tar xvfz node_exporter-* && cd node_exporter-* && ./node_exporter &`
   - Add to Prometheus: targets: ['<jenkins-ip>:9100']
2. Blackbox Exporter (Website Monitoring):
   - Download: `wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz`
   - Extract & Run: Similar to above, on port 9115.
   - Config blackbox.yml: modules for HTTP probes (e.g., probe application URL).
   - Add to Prometheus: job_name: 'blackbox', metrics_path: '/probe', params: module: [http_2xx], targets: ['<app-url>'].
3. Add Data Source in Grafana: Add Prometheus > Import Dashboards (ID 1860 for Node Exporter, 7587 for Blackbox).
4. Verify: In Prometheus UI (port 9090) > Targets > Ensure up. In Grafana > Dashboards > View metrics (CPU, RAM, website uptime).

**Troubleshooting:** Restart services after config changes: `killall prometheus && ./prometheus &`. Use `nohup` for background.

---

### Final Verification & Run
1. Commit code change to Git repo > Trigger Jenkins pipeline.
2. Access App: Via K8s service external IP.
3. Monitor in Grafana.
4. Receive email on build status.

This completes the project. 
