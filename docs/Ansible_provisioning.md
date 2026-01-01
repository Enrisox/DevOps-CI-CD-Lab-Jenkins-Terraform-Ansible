# ANSIBLE

**L'Inventory di Ansible** è fondamentalmente la "rubrica dei contatti" o il "database" dei server che vuoi gestire. <br>
**È il file dove elenchi gli indirizzi IP o i nomi a dominio (FQDN) delle macchine su cui Ansible deve andare a lavorare.**permette di organizzare l'infrastruttura in modo logico:

Senza un inventory, Ansible non saprebbe a chi inviare i comandi scritti nei Playbook.

- Host (Nodi): I singoli server (es. 192.168.1.91).
- Gruppi: Permettono di classificare i server per funzione (es. tutti i server web, tutti i database).
- Variabili: Puoi definire impostazioni specifiche per un singolo host o per un intero gruppo (es. la porta SSH o l'utente da usare).

## I formati più comuni

1. **Formato INI**
È il più semplice da leggere e scrivere, ideale per laboratori e configurazioni veloci.

```bash
[jenkins]
server-ci-01 ansible_host=192.168.1.91

[database]
db-prod ansible_host=192.168.1.50

[all:vars]
ansible_user=enrico
```
2. **Formato YAML**
Più moderno e strutturato, preferito in ambienti complessi perché segue la stessa sintassi dei Playbook.

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

**Inventory Statico vs Dinamico**

- **Statico**: Un file scritto a mano (come il tuo hosts.ini). Va bene quando hai pochi server che non cambiano spesso.
- **Dinamico**: Ansible interroga automaticamente un fornitore (es. AWS, Azure o anche il tuo Proxmox) per ottenere la lista aggiornata delle VM attive. È fondamentale quando le macchine vengono create e distrutte continuamente.

## Installazione Ansible su terminale wsl Ubuntu

Sull terminale Ubuntu:

```bash
sudo apt update
sudo apt install ansible -y
```
Verifica l'installazione:

```bash
ansible --version
```
La versione 2.16.3 di Ansible risulta correttamente installata sul sottosistema Linux

**Nel tuo home WSL:**
```bash
mkdir -p lab-ansible/{inventory,playbooks}
cd lab-ansible
```

**Struttura finale:**

lab-ansible/
├── inventory/
│   └── hosts.ini
└── playbooks/
    └── ping.yml



### scriviamo nel file L’INVENTORY

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

### PLAYBOOK DI TEST

```bash
nano playbooks/ping.yml

#INSERISCI DENTRO

- name: Test connectivity
  hosts: all
  gather_facts: false
  tasks:
    - name: Ping hosts
      ansible.builtin.ping:

```

```bash
ansible-playbook -i inventory/hosts.ini playbooks/ping.yml
```

IMMAGINE PING PONG NERA

**Ansible è operativo e pronto per il provisioning**
Ansible ha raggiunto entrambe le VM



- SSH con chiavi funziona
- Nessuna richiesta di password
- Autenticazione corretta
- Il modulo Ansible ping ha risposto

# BOOTSTRAP BASE e NETWORK CONFIGURATION DELLE VM clonate

Su ENTRAMBE le VM:

- hostname coerente
- pacchetti base
- sudo senza password (per automazione)

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
    # 1. Configurazioni di sistema (mentre la rete è ancora quella vecchia)
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

    # 2. Prepariamo il file di rete (non lo applichiamo ancora)
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

     #3. ULTIMO STEP: Cambiamo l'IP
    - name: Apply netplan (Il server cambierà IP e la connessione cadrà)
      ansible.builtin.shell: "netplan apply"
      async: 5
      poll: 0```

le due vm avevano già hostname fissato durante creazione con terraform dal clone 100
Used Ansible to enforce system state idempotently, including hostnames and sudo policies, regardless of initial VM configuration.


```bash
ansible-playbook -i inventory/hosts.ini playbooks/network-bootstrap.yml
```

**Questo è Infrastructure as Code**

## Per problematiche con chiavi SSH, da WSL terminal:

```bash
ssh-keygen -R 192.168.1.x
ssh-keyscan -H 192.168.1.x >> ~/.ssh/known_hosts
```

## STEP B

Su ENTRAMBE le VM:

- installare Docker Engine
- installare Docker Compose plugin
- permettere a enrico di usare Docker senza sudo

```bash
nano playbooks/install-docker.yml
```
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
**ESEGUI PLAYBOOK DOCKER**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/install-docker.yml
```

Terraform + Proxmox:

crea VM

collega NIC

NON gestisce netplan dentro la VM

💥 Terraform NON entra nel sistema operativo
💥 Terraform NON sa se l’interfaccia si chiama eth0 o ens18

👉 quello è Configuration Management
👉 quindi: Ansible

```bash
inventory/
  hosts.ini
group_vars/
  all.yml
host_vars/
  ci.yml
  runtime.yml
playbooks/
  bootstrap.yml
  network.yml   
  docker.yml
```

