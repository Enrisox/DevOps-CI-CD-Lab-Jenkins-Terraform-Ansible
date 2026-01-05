# Infrastructure as Code: Terraform Setup

To automate the creation of my virtual environment, I implemented Infrastructure as Code (IaC) using Terraform. This allowed me to clone and configure multiple VMs from my Ubuntu template with a single command.

1. **Terraform Installation** (WSL): I installed Terraform within my WSL Ubuntu environment to manage the infrastructure as code. I added the official HashiCorp GPG key and repository to ensure a secure and up-to-date installation.

```bash
#Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

#Add the official repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

#Install Terraform
sudo apt update && sudo apt install terraform -y
```

2. **Verification**: I confirmed the successful installation by checking the version:

```bash
terraform -version
#Output: Terraform v1.14.3 on linux_amd64
```

## Dedicated API Token in Proxmox
I began by creating a dedicated API Token in Proxmox (Datacenter -> Permissions -> API Tokens).

- User: root@pam
- Token ID: terraform-token

This allows Terraform to communicate with the Proxmox API securely without needing the master root password.

## Project Folder Preparation

Inside my WSL environment(Ubuntu), I organized the project workspace and initialized the Terraform providers:

```bash
mkdir ~/lab-proxmox && cd ~/lab-proxmox
touch provider.tf main.tf vars.tf
terraform init                     #Downloads the telmate/proxmox provider
```

## Defining the Infrastructure (main.tf)
I wrote the configuration to deploy two primary nodes: **the Jenkins CI server** and the **App Runtime agent**. **Both were cloned from my ubuntu-template (ID 100) and configured via Cloud-Init**.

