# 📦 Amazon Prime App — Freestyle CI/CD Pipeline on AWS
---------------------------------------------------------


## 📌 Project Overview

This project demonstrates a **Freestyle CI/CD pipeline** built on AWS Cloud.  
A Java-based web application is automatically built, stored as an artifact in S3, and deployed  
to Apache Tomcat — all triggered by a single GitHub code push with **zero manual steps**.

> ✅ Code Push → GitHub Webhook → Jenkins Freestyle Job → Maven Build → S3 Artifact → Tomcat Deployment

---

## 🏗️ Architecture Diagram

```
Developer
    │
    │  git push
    ▼
┌─────────────┐
│   GitHub    │  ── Webhook Trigger ──▶  ┌─────────────────┐
│  Repository │                          │  Jenkins Server  │
└─────────────┘                          │   (EC2 Linux)    │
                                         └────────┬────────┘
                                                  │
                                    ┌─────────────▼──────────────┐
                                    │       Maven Build           │
                                    │   (clean package → .war)   │
                                    └──────┬──────────┬──────────┘
                                           │          │
                                    ┌──────▼──┐  ┌────▼───────────┐
                                    │  AWS S3  │  │ Apache Tomcat  │
                                    │(Artifact │  │  (EC2 Linux)   │
                                    │ Storage) │  │  Port: 9090    │
                                    └─────────┘  └───────────────┘
```

---

## ⚙️ Tech Stack

| Category | Tool / Service | Purpose |
|---|---|---|
| Source Control | GitHub | Code repository & webhook trigger |
| CI/CD | Jenkins **Freestyle Job** | Pipeline automation |
| Build Tool | Maven | Compile & package Java app (.war) |
| Language | Java 21 | Application runtime |
| Web Server | Apache Tomcat | Application deployment (port 9090) |
| Cloud Compute | AWS EC2 (Amazon Linux) | Hosting Jenkins & Tomcat |
| Artifact Storage | AWS S3 | Store & version build artifacts |
| Security | AWS IAM Role | Secure, credential-free S3 access |
| Storage | AWS EBS (30GB) | EC2 persistent storage |
| Region | ap-south-1 (Mumbai) | AWS deployment region |

---

## 🔧 Infrastructure Setup

### EC2 Instance
- **AMI:** Amazon Linux 2
- **Storage:** 30 GB EBS
- **Region:** ap-south-1 (Mumbai)
- **IAM Role:** Attached for secure S3 access (no hardcoded credentials)

### Security Group Rules

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | SSH | Remote server access |
| 8080 | TCP | Jenkins Web UI |
| 9090 | TCP | Apache Tomcat App |

---

## 🛠️ Tools Installation

```bash
# Update system
sudo yum update -y

# Install Java 21
sudo yum install java-21-amazon-corretto -y

# Install Git
sudo yum install git -y

# Install Maven
sudo yum install maven -y

# Install Jenkins
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Install Apache Tomcat
wget https://downloads.apache.org/tomcat/tomcat-9/v9.x.x/bin/apache-tomcat-9.x.x.tar.gz
tar -xvzf apache-tomcat-9.x.x.tar.gz
```

---

## 🐱 Tomcat Configuration

### 1. Change Default Port (server.xml)
```xml
<!-- Changed from 8080 to 9090 to avoid conflict with Jenkins -->
<Connector port="9090" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

### 2. Configure User Roles (tomcat-users.xml)
```xml
<tomcat-users>
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <user username="admin" password="admin123"
        roles="manager-gui,manager-script"/>
</tomcat-users>
```

### 3. Update context.xml (Allow Remote Access)
```xml
<!-- Comment out the RemoteAddrValve in both files below -->
<!-- webapps/manager/META-INF/context.xml -->
<!-- webapps/host-manager/META-INF/context.xml -->

<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
```

### 4. Start Tomcat
```bash
cd apache-tomcat-9.x.x/bin
./startup.sh
```

---

## 🔁 Jenkins Freestyle Job Setup

### Step 1 — Initial Setup
```bash
# Get Jenkins initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step 2 — Install Required Plugins
Navigate to: `Manage Jenkins → Plugin Manager → Available`

- ✅ **S3 Publisher** — Upload artifacts to AWS S3
- ✅ **Deploy to Container** — Deploy .war to Tomcat

### Step 3 — Global Tool Configuration
Navigate to: `Manage Jenkins → Global Tool Configuration`
- Configure **JDK 21** path
- Configure **Maven** installation

### Step 4 — Create Freestyle Job (Prime App)

**Source Code Management:**
```
Repository URL: https://github.com/damodar-dev/amazon-prime-app.git
Branch: */master
```

**Build Trigger:**
```
✅ GitHub hook trigger for GITScm polling
```

**Build Step — Invoke top-level Maven targets:**
```
Goals: clean package
```

**Post-Build — S3 Upload:**
```
Bucket Name: your-s3-bucket-name
Source: **/*.war
Destination: artifacts/
```

**Post-Build — Deploy to Container:**
```
WAR/EAR files: **/*.war
Container: Tomcat 9.x
Manager URL: http://<EC2-PUBLIC-IP>:9090/manager/text
Credentials: admin/admin123
```

---

## 🔒 AWS IAM Role Setup (Security Best Practice)

Instead of using AWS Access Keys, an **IAM Role** is attached directly to the EC2 instance.

### IAM Role Policy (S3 Access)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

> ✅ Jenkins uploads artifacts to S3 **without any hardcoded credentials** — following AWS security best practices.

---

## 🔗 GitHub Webhook Setup

1. Go to your GitHub repo → **Settings → Webhooks → Add webhook**
2. Set **Payload URL:** `http://<JENKINS-EC2-IP>:8080/github-webhook/`
3. Content type: `application/json`
4. Select: **Just the push event**
5. Click **Add webhook**

> Every `git push` to master automatically triggers the Jenkins Freestyle job.

---

## 📦 CI/CD Pipeline Flow

```
1. Developer pushes code to GitHub (master branch)
         ↓
2. GitHub Webhook notifies Jenkins automatically
         ↓
3. Jenkins pulls latest code from GitHub
         ↓
4. Maven runs: mvn clean package
   → Compiles Java code
   → Runs tests
   → Packages into .war file
         ↓
5. .war artifact uploaded to AWS S3 bucket
   (secure access via IAM Role — no credentials needed)
         ↓
6. Jenkins deploys .war to Apache Tomcat (port 9090)
         ↓
7. Application is LIVE ✅
```

---

## ✅ Results & Outcomes

- ✅ Fully automated Freestyle pipeline — zero manual deployment steps
- ✅ Build artifacts versioned and stored in AWS S3
- ✅ Secure AWS access using IAM Roles (no hardcoded keys)
- ✅ GitHub Webhook triggers instant builds on every code push
- ✅ Apache Tomcat serves the application on port 9090
- ✅ Jenkins (8080) and Tomcat (9090) run on same EC2 without port conflicts

---

## 📁 Project Structure

```
amazon-prime-app/
├── src/
│   ├── main/
│   │   ├── java/          # Java source code
│   │   └── webapp/        # Web application files
│   └── test/              # Unit tests
├── pom.xml                # Maven build configuration
└── README.md              # Project documentation
```

---

## 🧠 Key Learnings

- Freestyle CI/CD job design and configuration in Jenkins
- AWS IAM Role-based access (security best practice)
- Artifact management & versioning with S3
- Jenkins plugin configuration (S3 Publisher, Deploy to Container)
- Apache Tomcat server administration & configuration
- GitHub Webhook integration for automated triggers
- Resolving port conflicts in multi-service environments

---

## 👨‍💻 Author

**S Damodararao**
DevOps & Cloud Engineer
📧 [damodarsampatirao@gmail.com](mailto:damodarsampatirao@gmail.com)
🔗 [LinkedIn](https://www.linkedin.com/in/sdamodararao/)
🐙 [GitHub](https://github.com/damodar-dev)
