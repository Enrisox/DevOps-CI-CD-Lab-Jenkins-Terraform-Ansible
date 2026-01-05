# About Me

My name is **Enrico Soci**, and I am a second-year student in the **ITS DevSecOps program(ECQ Level 5)** at the **Cisco Academy Olivetti**

I developed this project, together with other lab projects, as part of my learning path to gain hands-on experience with **DevSecOps tools and practices**, including infrastructure automation, CI/CD pipelines, containerization, and cloud integration.
This repository contains a self-hosted DevOps lab built to practice real-world CI/CD concepts using modern DevOps tools.
The lab environment is deployed on Proxmox and fully automated using Terraform and Ansible. CI/CD pipelines are implemented with Jenkins and Docker, leveraging AWS CodeCommit for source control and Amazon ECR for container image management.

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


## CI/CD Workflow

1. Application image is pushed to AWS ECR.
2. Jenkins pipeline is triggered.
3. Pipeline runs on the runtime agent.
4. Jenkins authenticates to AWS ECR.
5. Docker pulls the latest image.
6. Existing container is replaced with the new version.
7. Jenkins does not deploy via SSH.
8. The pipeline runs directly on the agent node, following a production-like model.


## Learning Objectives

1. Understand CI/CD pipelines and Jenkins agent architecture.
2. Practice Infrastructure as Code with Terraform.
3. Automate system configuration with Ansible.
4. Deploy containerized applications using Docker.
5. Integrate cloud services (AWS ECR) with on-prem infrastructure.

## About This Project

This lab represents my hands-on learning path toward a Junior DevOps role.
The focus is on fundamentals, automation, and clarity, rather than production-scale complexity.

**ENRICO SOCI**
