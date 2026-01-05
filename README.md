# DevSecOps Lab – Automated Infrastructure and CI/CD

## About Me

My name is **Enrico Soci**.  
I am a second-year student at the **ITS Academy Olivetti**, enrolled in the course **“Tecnico per la sicurezza e l’integrazione dei sistemi informativi – DevSecOps”**.
This project was developed as part of my training to practice and consolidate skills in **DevSecOps tools and methodologies**, including automation, infrastructure management, CI/CD pipelines, and containerized environments.

This repository contains a self-hosted DevOps lab built to practice real-world CI/CD concepts using modern DevOps tools.
The lab environment is deployed on Proxmox and fully automated using Terraform and Ansible. CI/CD pipelines are implemented with Jenkins and Docker, leveraging AWS CodeCommit for source control and Amazon ECR for container image management.

The focus is on fundamentals, automation, and clarity, rather than production-scale complexity.

## Hardware Setup

- Lenovo E73 .
- Quad-core CPU @ 2.70 GHz.
- 8 GB DDR3 RAM.
- SAMSUNG EVO 860 SSD 250 GB.

## Technologies Used

- Proxmox Virtual Environment
- Terraform 
- Ansible 
- Jenkins 
- Docker
- AWS CODECOMMIT
- AWS ECR
- Linux (Ubuntu server)

## Architecture

1. **Proxmox VE** – Hosts all virtual machines.
2. **Infrastructure as Code** – Terraform provisions VMs; Ansible bootstraps OS, networking, Docker, and users.
3. **CI/CD & Registry** – Jenkins (controller/agent) handles pipelines; AWS ECR stores Docker images.


## My CI/CD Workflow

1. **Code Push**: I push code updates to my AWS CodeCommit repository, which serves as the secure starting point for my automated pipeline.
2. **Jenkins Trigger**: I configured Jenkins to monitor my repository via SCM polling; it automatically detects my changes and triggers the automation process.
3. **Agent Execution**: My pipeline executes on a dedicated Jenkins Runtime Agent that I provisioned using Terraform and configured with Ansible on my Proxmox cluster.
4. **Security Scanning**: I integrated Trivy into the workflow to automatically perform a security audit of my Docker images, identifying vulnerabilities before any deployment occurs.
5. **Artifact Storage**: Once the build and security checks pass, I push the verified Docker image—my deployment artifact—to my private AWS ECR registry.
6. **Image Pull**: My runtime agent pulls the latest version of the application directly from AWS ECR, ensuring it always deploys the most recent verified image.
7. **Container Orchestration**: I manage the container lifecycle by automatically replacing the existing container with the new version, ensuring a clean and reliable update of the web app.
8. **Local Execution Model**: I designed the pipeline to run commands directly on the host machine via the Jenkins Agent; this follows a production-ready architecture that I built to avoid the security risks of SSH-based deployments.


## Learning Objectives

1. Understand **CI/CD pipelines** and **Jenkins agent** architecture.
2. Practice Infrastructure as Code with **Terraform**.
3. Automate system configuration with **Ansible**.
4. Deploy containerized applications using **Docker**.
5. Integrate cloud services (**AWS Codecommit and ECR**) with on-prem infrastructure.


**ENRICO SOCI**
