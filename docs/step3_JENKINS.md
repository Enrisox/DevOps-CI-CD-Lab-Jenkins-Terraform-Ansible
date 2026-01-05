# JENKINS Configuration and installation 
**What is Jenkins?** <br>

**Jenkins** is an automation server (typically self‑hosted) used mainly for CI/CD: you define a “pipeline” that executes builds, tests, scans, packaging, and deployment when an event occurs (push, PR, cron, manual) or when you decide.

**What Jenkins is for?**
1. Orchestrating pipeline steps (build/test/deploy) on one or more agents/nodes, even in parallel.
2. Integrating many different tools thanks to plugins (Git, Docker, Kubernetes, security scanners, notifications, etc.).
3. Keeping everything “in‑house” (homelab/on‑prem) and having total control over network, credentials, runners, caching, and environments.

**Difference compared to GitHub Actions**
- GitHub Actions is integrated into GitHub with managed or self‑hosted runners.
- Jenkins you usually manage yourself on a VM/container.
- Actions is native to GitHub; Jenkins is agnostic (GitHub/GitLab/Bitbucket/etc.), but you configure the integration yourself.

**Setup/management**: Actions is more “ready out of the box”; Jenkins requires more maintenance (updates, plugins, backups, security), but it is more flexible when you want custom environments.

**Note: I have previously implemented GitHub Actions with a self-hosted runner on an on-premises server in my previous DevOps project (available here on my GitHub). This experience provided me with a solid understanding of how self-managed runners interact with cloud-based CI/CD platforms.**   

https://github.com/Enrisox/Secure-Home-Lab-Docker

**Difference compared to similar AWS services (CodeBuild/CodePipeline)**
- with AWS you have managed services and pay per use; Jenkins is your own service.
- **Cloud integration:** AWS CI/CD is strongly integrated with IAM, ECR, ECS/EKS, S3, etc.; Jenkins can do it, but via plugin/credential management and configuration.
- **Portability**: Jenkins stays the same in homelab, on‑prem, or cloud; with AWS you are more “cloud‑native” but also more tied to the ecosystem.

## Jenkins deployment with Ansible

To ensure the reliability of my CI environment, I started by configuring a **persistent volume**. <br>
This means that if I delete the container or restart the VM, my Jenkins jobs, plugins, and configurations will not be lost because they are saved directly on the VM's filesystem at /home/enrico/jenkins_home


1)**I created the playbooks/deploy-jenkins.yml file to automate the entire setup**

```bash
nano playbooks/deploy-jenkins.yml
```

```bash
---
- name: Deploy Jenkins via Docker
  hosts: jenkins
  become: true

  tasks:
    - name: Ensure the data directory exists
      ansible.builtin.file:
        path: /home/userX/jenkins_home
        state: directory
        owner: 1000 # I used the standard Jenkins UID for correct permissions
        group: 1000
        mode: '0755'

    - name: Start the Jenkins container
      community.docker.docker_container:
        name: jenkins-server
        image: jenkins/jenkins:lts
        state: started
        restart_policy: always
        published_ports:
          - "8080:8080"   # Web Interface
          - "50000:50000" # Port for future build agents
        volumes:
          - /home/enrico/jenkins_home:/var/jenkins_home
        env:
          JAVA_OPTS: "-Djenkins.install.runSetupWizard=true"
```


## Ansible Galaxy
I used Ansible Galaxy, which is the official repository for Ansible roles and collections. A collection is a package that contains modules, plugins, and roles specific to a certain domain.

The community.docker collection is the official community-maintained package for managing Docker with Ansible. Since Ansible needs specific "modules" to communicate with the Docker API, I installed it within my WSL environment using the following command:
```bash
ansible-galaxy collection install community.docker      # Installing the Docker collection

ansible-playbook playbooks/deploy-jenkins.yml  # Running the Jenkins deployment playbook
```
This command downloads the latest modules for managing Docker containers, images, networks, volumes, and Compose files. It stores them in ~/.ansible/collections or within the project's collections/ folder.

## Post-Deployment: Accessing Jenkins

Once the playbook finished with an "OK" status, Jenkins was active and running. Here is how I proceeded with the initial setup:

1. Accessing the UI: I opened my browser and navigated to http://192.168.1.7:8080.
2. Unlocking Jenkins: A screen appeared asking for the Administrator Password.
3. Retrieving the Password: Instead of manually logging into the VM via SSH, I used an Ansible ad-hoc command from my WSL terminal to read the secret file:

```bash
ansible ci -a "cat /home/enrico/jenkins_home/secrets/initialAdminPassword"
```
This is a temporary file created by Jenkins during the first boot. It acts as a security measure to ensure that only the person with administrative access to the server filesystem can perform the initial configuration.

**Finalizing Setup**: I pasted the password into the browser and selected "Install suggested plugins" to complete the basic configuration.


# Agent Node (Runtime) Configuration

1. We will create an SSH key to allow Jenkins to enter machine .8.
2. We will configure machine .8 inside the Jenkins interface.
3. We will do a test: we will ask Jenkins to launch a command on .8 to see if it "obeys".
4. Now we will make it so that the "Foreman" (Jenkins on .7) can give orders to the "Worker" (Runtime on .8). To do this, we will use an SSH key.

The trick is this: we will generate the key on machine .7 (inside the Jenkins folder) and copy it to machine .8.

**1. Generate the SSH key on the Jenkins server**

I ran this command from my WSL. I used Ansible to tell machine .7 to create a key right in the folder I mapped for Jenkins:

```bash
ansible ci -m shell -a "ssh-keygen -t ed25519 -f /home/userX/jenkins_home/jenkins_key -N ''"
```
**2. Authorize Jenkins on the Runtime machine**
Now I took the "public key" just created and put it among the authorized keys of machine .8. Instead of doing manual copy-paste, I did everything in one go:

```bash
PUBLIC_KEY=$(ansible ci -a "cat /home/userX/jenkins_home/jenkins_key.pub" | grep ssh-ed25519) #it read the public key from .7

ansible runtime -m authorized_key -a "user=userX key='$PUBLIC_KEY'"   #I added it to .8 (Runtime)
```

## On the runtime VM

On the terminal of the Runtime VM I installed Java with these commands:

```bash
sudo apt update                        #updates packages
sudo apt install default-jre -y        #Installs Java (JRE)
java -version                          #verifies installation
```

## Node creation from Jenkins interface http://ip_VM_Jenkins(CI)

1. Go to Manage Jenkins > Nodes.
2. Click on + New Node on the left.
3. Enter the name runtime-node, select Permanent Agent and click Create.
4. Remote root directory: I wrote /home/userX/jenkins (the folder where Jenkins will work on the node).
5. Labels: I wrote runtime (it is used to tell future jobs to run exactly here).
6. Launch method: Select Launch agents via SSH.
7. Host: Static IP of the runtime VM: e.g.: 192.168.1.8.

**Credentials:**

1. Clicl  on Add > Jenkins.
2. Kind: select SSH Username with private key.
3. Username: userX.
4. Private Key: select Enter directly, then click Add and paste the private key you can see with this command:

```bash
ansible ci -a "cat /home/userX/jenkins_home/jenkins_key"
```

6. Click on "Add" 
7. I selected the credentials I just created from the Credentials dropdown menu.
8. Host Key Verification Strategy: I changed it to **Non verifying Verification Strategy** (to skip the manual check of the SSH fingerprint).
9. Save


NOTE: Why I chose to skip the host Key Verification?
I chose to skip this verification for three main reasons:

1. **Automation**: If I leave the verification active, the first time Jenkins tries to connect, it will fail because the SSH fingerprint of the Runtime VM is not yet in its "known_hosts" file. It would require a manual confirmation that cannot be done through the interface.
2. **Lab Environment**: In a development or homelab environment where VMs are often destroyed and recreated (for example, with Terraform or Proxmox clones), the SSH fingerprint changes every time. Skipping the check prevents the connection from breaking after every reinstall.
3. **Simplification**: It allows the Jenkins Controller to trust the Agent node immediately, ensuring the "Agent successfully connected" status without extra troubleshooting.

While this is perfect for a lab, in a production environment, you would typically use "Known hosts file Verification" to prevent Man-in-the-Middle (MITM) attacks.

**If in the node logs it says "Agent successfully connected and online" it is working**

If in the node logs it says "Agent successfully connected and online" it is working.
