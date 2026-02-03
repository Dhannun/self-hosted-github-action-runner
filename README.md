# 🤖 Self-Hosted GitHub Runner — Replication Guide (Ubuntu 22.04)

This document explains how to reproduce the self-hosted GitHub Actions runner used for:
- Docker builds  
- AWS ECR pushes  
- ArgoCD manifest updates  
- CI/CD automation for the SINNTS project.

------------------------------------------------------------

## 🎯 Goal

Set up an always-available GitHub Actions runner on AWS EC2 so CI jobs:
- start instantly (no queue)
- can build Docker images
- can authenticate to AWS ECR
- can update ArgoCD manifests

------------------------------------------------------------

## ✅ Prerequisites

You need:
- An AWS account
- A GitHub repository with Actions enabled
- SSH access to an EC2 instance

------------------------------------------------------------

## 🚀 Step 1 — Create EC2 Instance

Launch an EC2 with:

AMI: Ubuntu 22.04 LTS  
Instance: t3.medium (or t3.small minimum)  
Storage: 30–40GB  
Security Group: Allow SSH (22) from your IP  

Then SSH in:

ssh ubuntu@<YOUR-EC2-PUBLIC-IP>

------------------------------------------------------------

## 🐳 Step 2 — Install Docker (Required)

```shell
sudo apt update
sudo apt install -y docker.io unzip curl
sudo usermod -aG docker ubuntu
newgrp docker
```

Verify Docker:

```shell
docker ps
```

You should NOT need sudo.

------------------------------------------------------------

## ☁️ Step 3 — Install AWS CLI v2 (Ubuntu 22.04)

```shell
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify AWS CLI:

```shell
aws --version
```

Expected output: aws-cli/2.x.x ...

NOTE: Do NOT configure AWS credentials on this server. GitHub Actions will inject them via secrets.

------------------------------------------------------------

## 🏃 Step 4 — Register GitHub Self-Hosted Runner

On GitHub go to:

Repo → Settings → Actions → Runners → New self-hosted runner → Linux  

Then run the commands GitHub provides. They will look like this pattern:

```shell
mkdir actions-runner
cd actions-runner
curl -o runner.tar.gz -L <GITHUB_RUNNER_URL>
tar xzf runner.tar.gz
./config.sh --url <REPO_URL> --token <YOUR_TOKEN>
```

When asked for labels, enter:

self-hosted,docker,aws  

Start the runner:

```shell
./run.sh
```

------------------------------------------------------------

## 🔁 Step 5 — Make Runner Persistent (Survives Reboots)

```shell
sudo ./svc.sh install
sudo ./svc.sh start
```

------------------------------------------------------------

## 🔧 Step 6 — Update GitHub Workflow

In .github/workflows/staging-ci.yml change:

runs-on: ubuntu-22.04  

to:

runs-on: self-hosted  

Your pipeline will now run on your EC2.

------------------------------------------------------------

## 🧠 What This Enables

Every push to staging will:

1. Build Docker image on your EC2  
2. Push image to AWS ECR  
3. Update ArgoCD manifest  
4. Trigger Kubernetes deployment automatically via ArgoCD  

------------------------------------------------------------

## 📧 Optional — Add Email Notifications

Add this to the end of your workflow (YAML block to paste):

```yml
- name: Send deployment email
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.gmail.com
    server_port: 465
    username: ${{ secrets.EMAIL_USERNAME }}
    password: ${{ secrets.EMAIL_PASSWORD }}
    subject: "🚀 Staging Deployment Successful"
    body: |
      Staging deployment completed successfully.
      Image: ${{ secrets.ECR_URI }}:${{ env.BUILD_TAG }}
      Time: ${{ env.FORMATTED_DATE }}
    to: team@yourcompany.com
    from: ci-cd@yourcompany.com
```

------------------------------------------------------------

## 🛑 Troubleshooting

If runner is not showing in GitHub:
- Ensure `./run.sh is` running  
- Check GitHub → Settings → Actions → Runners  

If Docker permission error:
Run:
newgrp docker  

If AWS login failing:
Check:
```shell 
aws --version
```  

------------------------------------------------------------

## ✅ Summary

Hosted Runner:
- Has queues  
- Can fail due to capacity  

Self-Hosted Runner:
- No queue  
- Faster builds  
- More reliable  
- Full control  

------------------------------------------------------------

Maintained by: Yunus Muhammad Abudukhan <br>
Prepared: With ❤️ by [Dhannun](https://github.com/Dhannun)  
Purpose: Reliable CI/CD for staging environment  
