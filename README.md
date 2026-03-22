# 🎨 Pix-Mix — Production DevOps Setup

A static HTML/CSS/JS web application deployed on AWS with a full CI/CD pipeline, Blue-Green deployment strategy, and CloudWatch monitoring.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture Diagram](#architecture-diagram)
- [Setup Instructions](#setup-instructions)
- [CI/CD Pipeline](#cicd-pipeline)
- [Blue-Green Deployment](#blue-green-deployment)
- [CloudWatch Monitoring](#cloudwatch-monitoring)

---

## 📌 Project Overview

**Pix-Mix** is a browser-based photo creation and editing application built with HTML, CSS, and JavaScript. It allows users to create and edit their own photos directly in the browser — no installation required. The app is served via Nginx inside a Docker container and deployed on AWS Elastic Beanstalk using a Blue-Green deployment strategy, fully automated with GitHub Actions.

| Technology | Purpose |
|-----------|---------|
| HTML/CSS/JS | Frontend application |
| Nginx | Static file web server |
| Docker | Containerization |
| AWS ECR | Docker image registry |
| AWS Elastic Beanstalk | Application hosting |
| GitHub Actions | CI/CD automation |
| AWS CloudWatch | Monitoring & alerting |
| AWS SNS | Email notifications |

---

## 🏗️ Architecture Diagram

```
Developer (Windows)
        │
        │  git push
        ▼
┌───────────────────┐
│    GitHub Repo    │
└────────┬──────────┘
         │ triggers
         ▼
┌───────────────────────────────────────┐
│         GitHub Actions Pipeline       │
│                                       │
│  1. Checkout code                     │
│  2. Configure AWS credentials         │
│  3. Login to Amazon ECR               │
│  4. Build Docker image                │
│  5. Push image to ECR                 │
│  6. Deploy to Elastic Beanstalk       │
└───────────────┬───────────────────────┘
                │
                ▼
┌───────────────────────────────────────┐
│           AWS ECR                     │
│   (stores Docker images)              │
│   123456.dkr.ecr.us-east-1/my-app    │
└───────────────┬───────────────────────┘
                │
                ▼
┌───────────────────────────────────────┐
│       AWS Elastic Beanstalk           │
│                                       │
│  ┌─────────────┐  ┌─────────────┐    │
│  │  Blue Env   │  │  Green Env  │    │
│  │  (idle) 💤  │  │  (live) ✅  │    │
│  └─────────────┘  └─────────────┘    │
│         └──────────────┘             │
│              URL Swap ⚡              │
└───────────────┬───────────────────────┘
                │
                ▼
┌───────────────────────────────────────┐
│         AWS CloudWatch                │
│                                       │
│  • CPU Utilization Dashboard          │
│  • HTTP Error Monitoring              │
│  • SNS Email Alerts                   │
└───────────────────────────────────────┘
```

---

## ⚙️ Setup Instructions

### Prerequisites
- AWS Account
- GitHub Account
- Docker Desktop (optional for local testing)

### 1. Clone the repository
```bash
git clone https://github.com/yourusername/pix-mix.git
cd pix-mix
```

### 2. Configure GitHub Secrets
Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** and add:

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Your AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Your AWS secret key |
| `AWS_REGION` | AWS region e.g. `us-east-1` |
| `ECR_REPOSITORY` | ECR repository name e.g. `my-app` |
| `EB_APP_NAME` | Elastic Beanstalk app name e.g. `my-app` |
| `EB_ENV_NAME` | Target environment e.g. `my-app-green` |

### 3. AWS Setup
- Create an **ECR repository** named `my-app`
- Create two **Elastic Beanstalk environments**:
    - `my-app-blue` (Docker platform)
    - `my-app-green` (Docker platform)
- Attach `AmazonEC2ContainerRegistryReadOnly` policy to `aws-elasticbeanstalk-ec2-role`

### 4. Push to GitHub
```bash
git add .
git commit -m "Initial deployment"
git push origin main
```
The pipeline will trigger automatically! ✅

---

## 🚀 CI/CD Pipeline

The pipeline is defined in `.github/workflows/deploy.yml` and triggers on every push to the `main` branch.

### Pipeline Steps

```
Push to main
      ↓
Checkout code
      ↓
Configure AWS credentials
      ↓
Login to Amazon ECR
      ↓
Build Docker image
      ↓
Push image to ECR (tagged with git SHA)
      ↓
Generate Dockerrun.aws.json
      ↓
Deploy to Elastic Beanstalk (idle environment)
```

### Dockerfile
```dockerfile
# Use Nginx to serve static files
FROM nginx:alpine

# Remove default nginx page
RUN rm -rf /usr/share/nginx/html/*

# Copy app files into nginx
COPY . /usr/share/nginx/html

# Expose port 80
EXPOSE 80
```

---

## 🔵🟢 Blue-Green Deployment

Blue-Green deployment ensures **zero downtime** when releasing new versions.

### How it works

```
┌─────────────────────────────────────────┐
│                                         │
│   Users                                 │
│     │                                   │
│     ▼                                   │
│   Live URL                              │
│     │                                   │
│     ├──→ Blue Environment (idle 💤)     │
│     └──→ Green Environment (live ✅)    │
│                                         │
└─────────────────────────────────────────┘
```

### Deployment Cycle

1. **New update pushed** → deploys to idle environment (e.g. Blue)
2. **Test** the new version on the idle environment URL
3. **Swap URLs** in Elastic Beanstalk (Actions → Swap environment URLs)
4. **Idle becomes live** with zero downtime ⚡
5. **Previous live becomes idle** — acts as instant rollback if needed
6. **Repeat** on next deployment ♻️

### Rollback
If something goes wrong after a swap:
1. Go to Elastic Beanstalk
2. Click **Actions** → **Swap environment URLs** again
3. Traffic instantly returns to the previous version ⚡

---

## 📊 CloudWatch Monitoring

### Dashboard
A CloudWatch dashboard `pix-mix-dashboard` provides real-time visibility into:
- **CPU Utilization** — line chart showing server load over time
- **Alarm Status** — live status of all configured alarms

### Alarms

| Alarm | Metric | Threshold | Action |
|-------|--------|-----------|--------|
| `pix-mix-high-cpu` | CPUUtilization | > 80% for 5 mins | Email alert |
| `pix-mix-http-errors` | HTTP 5xx errors | > 5 in 5 mins | Email alert |

### Alarm States

| State | Meaning |
|-------|---------|
| 🟢 **OK** | Everything is healthy |
| 🟡 **Insufficient data** | Not enough data yet (normal after creation) |
| 🔴 **In alarm** | Threshold breached — check your email! |

### Notifications
Email alerts are sent via **AWS SNS** topic `pix-Mix-Alerts` to your registered email address whenever an alarm is triggered.

---

## 📁 Project Structure

```
pix-mix/
 ├── .github/
 │    └── workflows/
 │         └── deploy.yml        ← GitHub Actions pipeline
 ├── css/
 │    └── app.css                ← Application styles
 ├── js/
 │    ├── app.js                 ← Main JavaScript
 │    └── app_alt.js             ← Alternative JavaScript
 ├── images/
 │    ├── peace.jpg
 │    └── placeholder.png
 ├── cover.css                   ← Cover styles
 ├── index.html                  ← Main entry point
 ├── Dockerfile                  ← Docker configuration
 ├── Dockerrun.aws.json          ← Elastic Beanstalk config
 └── README.md                   ← This file
```

---

## 👨‍💻 Author

Built as part of a DevOps learning journey covering Docker, AWS, CI/CD, and monitoring best practices.