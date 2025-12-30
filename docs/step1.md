# Installazione Proxmox 

1) installato proxmox da chiavetta usb contenete ISO , F1 per entrare nel Bios e impostare chiavetta usb come metodo di boot principale.
2) installare graphical mode, scegli ip statico e hostname e password di root..
3) apt update e apt upgrade : darà errore per colpa della licenza mancante quindi bisogna cancellare o commentare contenuto sia di apt/sources.list.d/ceph.sources che di  /etc/apt/sources.list.d/pve-enterprise.sources
4) nano /etc/apt/sources.list.d/pve-no-subscription.sources   e incolla dentro

```bash
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Architectures: amd64
```

5)aggiungiamo nuovo utente sul server con comando linux adduser e aggiungiamo utente con stesso nome e password anche sulla gui di proxmox , raggiungibile a ip:8006
6)creo vm ubuntu server con 1 core e 2 gb di ram.. 
7)finita istallazione procedo a configurazione rete.. 


No: se tu hai “la 24.04” tra le ISO, non è quella giusta per questo metodo. Per cloud-init “fatto bene” ti serve la cloud image (…cloudimg-amd64.img), non la ISO di installazione.
8)scarico da url la iso cloudimg in local su proxmox

8)creare è un Cloud-Init drive. Nel mondo dell'automazione (Terraform/Ansible), è il "pezzo del puzzle" fondamentale.

Senza un drive Cloud-Init, quando Terraform crea una VM su Proxmox, la VM si accenderebbe ma rimarrebbe ferma alla schermata di installazione o di login, obbligandoti a intervenire a mano. Con il drive Cloud-Init, Terraform può "iniettare" automaticamente l'utente, la password, le chiavi SSH e l'indirizzo IP.
```bash
qm create 100 --name ubuntu-template --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci


VMID=100
STORAGE=local-lvm
IMG=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img

qm set $VMID --scsi0 ${STORAGE}:0,import-from=$IMG
qm set $VMID --ide2 ${STORAGE}:cloudinit
qm set $VMID --boot order=scsi0
qm set $VMID --serial0 socket --vga serial0
```



9)da powershell su windows installo sottosistema linux WLS. 
```bash
wsl --install
```
10)genera chiavi ssh pc casa e mostrala 
```bash
ssh-keygen -t ed25519 -C "PC-CASA"
cat ~/.ssh/id_ed25519.pub
```
11)copiala tutta, anche commento

12) configura cloud init con username, password, chiavi ssh generate prima, come dns domain :lan e come dns name 1.1.1.1
13) ip:dinamico
14) controlla la vm 100 avviandola e se tutto è ok: clicca regenerate image in alto a dx
15) spegni vm 100
16) tasto dx su vm 100, Cerca la voce "Convert to template" (di solito è verso la fine del menu).

Clicca su "Yes" per confermare.

Cosa deve succedere ora:

L'icona cambierà: non sarà più un quadratino con uno schermo blu/grigio, ma diventerà un foglietto con una corona dorata/gialla.

Se provi a cliccare sul tasto "Start", vedrai che è disattivato. È giusto così: un template è uno stampo, non si accende mai, si usa solo per far nascere altre VM
17) Il prossimo passo è installare Terraform nel tuo sottosistema Linux (WSL) e preparare il collegamento con Proxmox.

1. Installare Terraform nel WSL
Apri il tuo terminale Ubuntu/WSL su Windows e incolla questi comandi (uno alla volta) per installare Terraform:
```bash
# Aggiungi la chiave di HashiCorp
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list        # Aggiungi il repository ufficiale

sudo apt update && sudo apt install terraform -y    # Installo Terraform
```
**controllo che sia installato**
```bash
terraform -version
```

Terraform v1.14.3
on linux_amd64

**creato l'API Token su Proxmox (Datacenter -> Permissions -> API Tokens). Senza quello, Terraform non ha il permesso di entrare nel server.**

- Clicca su Add.
- User: root@pam
- Token ID: terraform-token
- Privilege Separation: Deselezionalo (per ora semplifica la gestione dei permessi).
- Clicca su Add.

**Copia subito il "Secret" che appare (una stringa lunga). Non verrà più mostrato. Copialo in un file di testo temporaneo sul tuo Windows.**

-------------------------------------------
**Preparare la cartella del progetto
Sempre nel WSL, crea una cartella per il tuo progetto e i file necessari:**

```bash
mkdir ~/lab-proxmox && cd ~/lab-proxmox
touch provider.tf main.tf vars.tf
```
-----------------------------------
**Scrivere il file provider.tf**
```bash
nano provider.tf
```
**Inizializzare Terraform scaricando driver necessari**

```bash
terraform init
```

foto con scritte verdi vuol dire che ha funzionato!!

```bash
nano provider.tf

terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.11"
    }
  }
}

provider "proxmox" {
  pm_api_url          = "https://IP-PROXMOX:8006/api2/json"
  pm_api_token_id     = "root@pam!terraform-token"
  pm_api_token_secret = "TOKEN-API-GENERATO-PRIMA"
  pm_tls_insecure     = true # Se non hai certificati SSL validi
  pm_parallel         = 1   #IMPORTANTE SE NON DA ERRORE COI PERMESSI 
}


```
su gui proxmox, permessi
1. root@pam permessi di / , gruppo admins
2. permessi token API --aggiungi--> token id = terraform-token   utente root@pam , permessi administrator in /

```bash
nano main.tf
```
**incolla dentro**
```bash
# --- VM 1: JENKINS ---
resource "proxmox_vm_qemu" "jenkins_vm" {
  name        = "jenkins-ci"
  target_node = "Proxmox"
  vmid        = 101
  clone       = "100"
  full_clone  = true

  cores   = 2
  memory  = 4096
  agent   = 1

  disk {
    slot    = 0
    size    = "20G"
    type    = "scsi"
    storage = "local-lvm"
  }

  os_type   = "cloud-init"
  ipconfig0 = "ip=192.168.1.91/24,gw=192.168.1.1"

  ciuser     = "enrico"
  sshkeys    = <<EOF
  ssh-ed25519**********chiave-ssh*************
  EOF
}

# --- VM 2: RUNTIME ---
resource "proxmox_vm_qemu" "runtime_vm" {
  name        = "app-runtime"
  target_node = "Proxmox"
  vmid        = 102
  clone       = "100"
  full_clone  = true

  cores   = 2
  memory  = 4096
  agent   = 1

  disk {
    slot    = 0      # <--- Deve essere una stringa completa
    size    = "20G"
    type    = "scsi"       
    storage = "local-lvm"
  }

  os_type   = "cloud-init"
  ipconfig0 = "ip=192.168.1.92/24,gw=192.168.1.1"

  ciuser     = "enrico"
  sshkeys    = <<EOF
  ssh-ed25519**********chiave-ssh*************
  EOF
}
```
**testiamo main.tf**
```bash
terraform plan
```
**diamo vita alle due vm**
```bash
terraform apply
```
