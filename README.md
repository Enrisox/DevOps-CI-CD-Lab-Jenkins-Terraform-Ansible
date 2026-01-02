# Proxmox-CI/CD Lab

This repository contains a self-hosted DevOps lab built to practice real-world CI/CD concepts using modern DevOps tools.

The entire environment is deployed on Proxmox and automated with Terraform and Ansible, while application delivery is handled through Jenkins, Docker, and AWS ECR.

**Hardware Setup**

Host machine
Lenovo E73 
Quad-core CPU @ 2.9 GHz
8 GB DDR3 RAM

**Technologies Used**

- Proxmox VE 
- Terraform 
- Ansible 
- Jenkins 
- Docker
- AWS ECR
- Linux (Ubuntu)


**Architecture**

1. Proxmox VE
   Hosts all virtual machines
2. Infrastructure as Code
   Terraform provisions VMs from templates
   Ansible bootstraps OS, networking, Docker, and users
3. **Jenkins Controller**: Manages pipelines, jobs, and credentials <br>
**Jenkins Agent** (runtime node): Runs Docker and executes deployment steps locally
4. Container Registry: AWS ECR stores application Docker images

**CI/CD Workflow**

1. Application image is pushed to AWS ECR
2. Jenkins pipeline is triggered
3. Pipeline runs on the runtime agent
4. Jenkins authenticates to AWS ECR
5. Docker pulls the latest image
6. Existing container is replaced with the new version
7. Jenkins does not deploy via SSH.
8. The pipeline runs directly on the agent node, following a production-like model.


**Learning Objectives**

1. Understand CI/CD pipelines and Jenkins agent architecture
2. Practice Infrastructure as Code with Terraform
3. Automate system configuration with Ansible
4. Deploy containerized applications using Docker
5. Integrate cloud services (AWS ECR) with on-prem infrastructure

**About This Project**

This lab represents my hands-on learning path toward a Junior DevOps role.
The focus is on fundamentals, automation, and clarity, rather than production-scale complexity.
