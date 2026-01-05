# Ansible configuration

**Ansible's Inventory** is the file/list that defines which machines/servers Ansible manages. <br>
It is the file where I list the IP addresses or Fully Qualified Domain Names (FQDN) of the machines on which Ansible must work. It allows me to organize the infrastructure logically:

Without an inventory, Ansible wouldn't know where to send the commands written in the Playbooks.

- **Hosts** (Nodes): The individual servers (e.g., 192.168.1.91).
- **Groups**: These allow me to classify servers by function (e.g., all web servers, all databases).
- **Variables**: I can define specific settings for a single host or an entire group (e.g., the SSH port or the user to be used).

## Most common formats
**INI Format**  is the simplest to read and write, ideal for labs and quick configurations.

```bash
[jenkins]
ci ansible_host=192.168.1.7

[runtime_group]
runtime ansible_host=192.168.1.8

[all:vars]
ansible_user=enrico
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```


**YAML Format** More modern and structured, preferred in complex environments because it follows the same syntax as Playbooks.

```yaml
all:
  hosts:
    server-ci-01:
      ansible_host: 192.168.1.91
  children:
    jenkins:
      hosts:
        server-ci-01:
```

### Static vs. Dynamic Inventory

- **Static**: A manually written file (like my hosts.ini). It works well when you have a few servers that don't change often.
- **Dynamic**: Ansible automatically queries a provider (e.g., AWS, Azure, or even my Proxmox) to obtain an updated list of active VMs. It is essential when machines are constantly being created and destroyed.

## Ansible Installation on WSL Ubuntu terminal
On the Ubuntu terminal:

```bash
sudo apt update
sudo apt install ansible -y

ansible --version     #Verify installation
```
Ansible version 2.16.3 is correctly installed on the Linux subsystem.

**In WSL:**
```bash
mkdir -p lab-ansible/{inventory,playbooks}
cd lab-ansible
```

**Final structure:**

lab-ansible/
├── inventory/
│   └── hosts.ini
└── playbooks/
    └── 


## Writing the INVENTORY file "hosts.ini"

```bash
nano inventory/hosts.ini
```
```bash
[jenkins]
ci ansible_host=192.168.1.7

[runtime_group]
runtime ansible_host=192.168.1.8

[all:vars]
ansible_user=enrico
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

## TEST PLAYBOOK

```bash
nano playbooks/ping.yml
```

```bash
- name: Test connectivity
  hosts: all
  gather_facts: false
  tasks:
    - name: Ping hosts
      ansible.builtin.ping:
```
Finally, I executed the playbook to verify the connection:
```bash
ansible-playbook -i inventory/hosts.ini playbooks/ping.yml
```

IMMAGINE PING PONG NERA

Ansible is operational and ready for provisioning Ansible has successfully reached both VMs.

- SSH with keys is working correctly.
- No password prompts during connection.
- Authentication is successful.
- The Ansible ping module responded as expected.

## BASE BOOTSTRAP and NETWORK CONFIGURATION of Cloned VMs
On BOTH VMs, I aimed to achieve:

- Coherent hostnames
- Installation of base packages.
- Sudo access without a password (essential for automation).

**I created the bootstrap playbook:**
```bash
nano playbooks/bootstrap.yml
```

```bash
- name: Network & Hostname Bootstrap
  hosts: all
  become: true

  vars:
    net_interface: eth0
    net_gateway: 192.168.1.1
    net_dns: [1.1.1.1, 8.8.8.8]
    host_ips:
      ci: 192.168.1.7
      runtime: 192.168.1.8

  tasks:
    # 1. System configurations (while the old network is still active)
    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ inventory_hostname }}"

    - name: Ensure sudo without password for enrico
      ansible.builtin.copy:
        dest: /etc/sudoers.d/enrico
        content: "enrico ALL=(ALL) NOPASSWD:ALL\n"
        mode: "0440"

    - name: Install base packages
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - git
          - gnupg
          - lsb-release
        state: present
        update_cache: yes

    # 2. Prepare the network file (not applied yet)
    - name: Configure netplan
      ansible.builtin.copy:
        dest: /etc/netplan/01-ansible.yaml
        owner: root
        group: root
        mode: '0644'
        content: |
          network:
            version: 2
            renderer: networkd
            ethernets:
              {{ net_interface }}:
                dhcp4: no
                addresses:
                  - "{{ host_ips[inventory_hostname] }}/24"
                routes:
                  - to: default
                    via: "{{ net_gateway }}"
                nameservers:
                  addresses: {{ net_dns | to_json }}

     # 3. FINAL STEP: IP Change
    - name: Apply netplan (The server will change IP and the connection will drop)
      ansible.builtin.shell: "netplan apply"
      async: 5
      poll: 0
```

le due vm avevano già hostname fissato durante creazione con terraform dal clone 100
Used Ansible to enforce system state idempotently, including hostnames and sudo policies, regardless of initial VM configuration.

**I executed the playbook with the following command:**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/network-bootstrap.yml
```
Although the two VMs already had their hostnames set during the Terraform creation process from the ID 100 clone, I used Ansible to enforce the system state **idempotently**. This ensures that hostnames and sudo policies remain correct regardless of the initial VM configuration.

### what is Idempotentcy?**
**Terraform without idempotency:**
terraform apply → creates 3 EC2
terraform apply → creates 3 MORE EC2 (6 total!)
  
**Terraform with idempotency(idempotent):**
terraform apply → creates 3 EC2  
terraform apply → 3 EC2 still there → "no changes"
<br>
**Questo è Infrastructure as Code**



## STEP B

On BOTH VMs, my objectives were:

1. Install Docker Engine.
2. Install the Docker Compose plugin.
3. Allow the user enrico to use Docker without needing sudo.

**I created the Docker installation playbook:**

```bash
nano playbooks/install-docker.yml
```
**Playbook content:**

```bash
- name: Install Docker
  hosts: all
  become: true

  tasks:
    - name: Remove old Docker packages
      ansible.builtin.apt:
        name:
          - docker
          - docker-engine
          - docker.io
          - containerd
          - runc
        state: absent

    - name: Install Docker prerequisites
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
        state: present
        update_cache: yes

    - name: Add Docker GPG key
      ansible.builtin.shell: |
        mkdir -p /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Add Docker repository
      ansible.builtin.shell: |
        echo \
          "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
          https://download.docker.com/linux/ubuntu \
          $(lsb_release -cs) stable" \
          > /etc/apt/sources.list.d/docker.list
      args:
        creates: /etc/apt/sources.list.d/docker.list

    - name: Install Docker Engine
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: yes

    - name: Ensure Docker service is running
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Add enrico to docker group
      ansible.builtin.user:
        name: enrico
        groups: docker
        append: yes
```
**Executing Docker playbook**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/install-docker.yml
```
## Ansible Configuration (ansible.cfg)
I created the ansible.cfg file to act as the central "command center" for the project's settings. This file is essential for streamlining automation, as it eliminates the need to manually pass repetitive arguments (such as the inventory path or the remote user) every time a command is executed.

**1. ansible.cfg creation**
By placing this file in the project root (~/lab-ansible/), I ensured that the environment is consistent and portable. Ansible automatically prioritizes a local .cfg file over global system settings.
```bash
nano ansible.cfg
```
Configuration Content:

```ini
[defaults]
inventory = inventory/hosts.ini
remote_user = userX
```

**2. Validation & Results**
With the configuration in place, I verified the entire setup by querying both nodes simultaneously to check the Docker status:

```bash
ansible all -a "docker ps"      #instead of: ansible all -i inventory/hosts.ini -a "docker ps"
```

### Key Objectives Achieved:
1. Automation: I successfully deployed the Docker Engine, CLI, and Compose plugins across both servers at once using a single playbook.
2. Permission Hardening: I added userX to the docker group, allowing for container management without sudo, which is a requirement for seamless CI/CD integration.
3. System Verification: I confirmed via Ansible that the Docker service is active, enabled on boot, and responding correctly on all lab nodes.
4. The ansible.cfg file makes the project self-contained. If the project folder is moved or shared, the settings for the inventory and the remote user remain intact, ensuring that the playbooks always run in the same context.


**Why i used the shorter command?**
Once the ansible.cfg file is configured, the command becomes much simpler because Ansible now has a "default" behavior.

Default Inventory: Since the path inventory/hosts.ini is defined inside ansible.cfg, Ansible automatically loads it. This makes the -i flag redundant.



  
