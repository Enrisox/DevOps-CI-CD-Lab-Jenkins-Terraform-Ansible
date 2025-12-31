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
[ci]
IP
[runtime]
IP
[all:vars]
ansible_user=enrico
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3

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

