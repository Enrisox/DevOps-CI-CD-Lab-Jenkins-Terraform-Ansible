# Proxmox Installation & Initial Setup

1. **Hardware Boot**: I installed Proxmox using a USB flash drive containing the ISO image. I accessed the BIOS by pressing F1 and configured the USB drive as the primary boot device.
2. **System Configuration**: I proceeded with the graphical installation mode, where I assigned a static IP address, defined the system hostname, and set the root password.
3. **Repository Cleanup**: After the first boot, I ran apt update and apt upgrade. To resolve errors caused by the lack of a commercial license, I deleted the enterprise-only repository entries in both /etc/apt/sources.list.d/ceph.sources and /etc/apt/sources.list.d/pve-enterprise.sources.
4. **No-Subscription Repository**: I then configured the community-supported repository by creating a new file with nano /etc/apt/sources.list.d/pve-no-subscription.sources and adding the appropriate repository URL to enable system updates.
```bash
nano /etc/apt/sources.list.d/pve-no-subscription.sources 
```

```bash
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Architectures: amd64
```
## User Management & VM Template Creation

1. **User Synchronization**: I created a new system user using the adduser command. To ensure seamless access, I replicated the same username and password within the Proxmox GUI (accessible via https://[IP]:8006).
2. **VM Provisioning**: I provisioned a new Ubuntu Server VM, allocating 1 CPU core and 2 GB of RAM to meet the application's runtime requirements.
3. **Network Setup**: After the base installation, I manually configured the network settings to ensure the VM was reachable within my lab environment.
4. **Cloud-Init Image Acquisition**: I downloaded the Ubuntu Cloud-Init (cloudimg) ISO directly to the local Proxmox storage to serve as the foundation for my automation templates.

## Automating with Cloud-Init

I implemented a **Cloud-Init drive**, which is the essential for my Terraform and Ansible automation workflow. Without a Cloud-Init drive, any VM created by Terraform would remain stuck at the initial login or installation screen, requiring manual intervention. By integrating Cloud-Init, I enabled Terraform to automatically "inject" critical configurations during the first boot, including:

- **SSH Key injection**: I implemented this for secure, passwordless remote access, allowing tools like Ansible to automate tasks without manual intervention.
- **User account & password**: I configured a primary user with a password as a fallback for local console access and to authorize administrative (sudo) operations.
- **Static IP** address assignment.


## Template Creation Commands
I used the QEMU Manager (qm), the native Proxmox CLI utility, to provision the base templates. This approach allowed me to script the initial VM configuration, ensuring a repeatable and consistent baseline for the subsequent Terraform-led automation.

```bash
qm create 100 --name ubuntu-template --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci          #Created the VM template shell

#Define variables for the import process
VMID=100
STORAGE=local-lvm
IMG=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img

#Import the disk image and set up the Cloud-Init drive
qm set $VMID --scsi0 ${STORAGE}:0,import-from=$IMG
qm set $VMID --ide2 ${STORAGE}:cloudinit
qm set $VMID --boot order=scsi0
qm set $VMID --serial0 socket --vga serial0
```

## Workstation Setup & SSH Key Generation

1. **WSL Installation**: I installed the Windows Subsystem for Linux (WSL) on my Windows machine via PowerShell to create a native Linux environment for my DevOps tools.

```bash
wsl --install
```

2. **SSH Key Pair**: I generated a high-security ED25519(better than RSA) SSH key pair to establish a secure connection between my workstation and the lab environment.

```bash
ssh-keygen -t ed25519 -C "PC-HOME"       #-C is the label
```

3. **Display the public key to be copied**

```bash
cat ~/.ssh/id_ed25519.pub
```

4. **Key Management**: I copied the entire public key string, including the comment, to integrate it into the Cloud-Init configuration

## Cloud-Init Configuration & Template Conversion

1. **Cloud-Init Customization**: I configured the Cloud-Init parameters with my designated username, password, and the previously generated SSH key. I also set the DNS domain to .lan and the DNS server to 1.1.1.1.
2. **IP Assignment**: For the template baseline, I selected a Dynamic IP (DHCP) to ensure maximum flexibility when cloning new instances.

NOTE: I configured the base template with a Dynamic IP (DHCP) to ensure it remains a generic and reusable 'gold image'. However, during the deployment phase, I used Terraform to inject Static IPs into each instance. This ensures that critical infrastructure components—like the Monitoring stack (.7) and the Jenkins Agent (.8)—always reside at fixed addresses for reliable communication

3. **Validation**: I performed a test boot of VM 100 to verify the settings. Once confirmed, I used the "Regenerate Image" function in the Proxmox UI to finalize the Cloud-Init drive.
4. **System Shutdown**: I safely powered down VM 100 to prepare it for conversion.
5. **Proxmox Template Conversion**: I converted the VM into a Proxmox Template.

**Result: The VM icon changed to the Template icon, and the "Start" button was disabled.**
**Purpose**: This template now acts as a "gold image", ensuring that all future VMs are identical and ready for automation.

## Infrastructure as Code: Terraform Setup

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




------------------------------------------------------------------------------------------
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
# Scrivere il file provider.tf
Preparazione 
1. Installa Terraform: Sul tuo WSL (o Linux).
2. Crea la cartella di progetto: Una cartella pulita (es. lab-proxmox) per ogni progetto.
3. Scrittura (I file .tf)Puoi chiamarli come vuoi, ma lo standard è:provider.tf: Dove dici a Terraform con chi parlare (Proxmox, AWS, Azure) e gli dai le chiavi (Token API).main.tf: Dove descrivi cosa vuoi creare (le VM, i dischi, la rete).
4. I 3 Comandi Magici nel terminale, dentro la cartella dei file, esegui sempre questa sequenza:
5. **terraform init** = Scarica i "driver" (i plugin) necessari per parlare con Proxmox.
6. **terraform plan** = Ti mostra cosa farebbe senza toccare nulla.Legge il progetto e ti dice quanto costa e cosa rompe
7. **terraform apply** = Esegue fisicamente i cambiamenti su Proxmox.

## Inizializzare Terraform scaricando driver necessari

```bash
terraform init
```

foto con scritte verdi vuol dire che ha funzionato!!

```bash
nano provider.tf
```
```bash
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.11"
    }
  }
}

provider "proxmox" {
  pm_api_url      = "https://192.168.1.53:8006/api2/json"
  pm_user         = "enrico@pve"
  pm_password     = "password"
  pm_tls_insecure = true
  pm_parallel     = 1

}

```

```
## gestire permessi
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
  clone       = "ubuntu-template"         #vuole nome,
  full_clone  = true
  scsihw      = "virtio-scsi-pci"

  cores   = 2
  memory  = 4096
  balloon = 0
  agent   = 0

  disk {
    slot    = 0
    size    = "20G"
    type    = "scsi"
    storage = "local-lvm"
  }
  network {
    model  = "virtio"
    bridge = "vmbr0" 
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
  clone       = "ubuntu-template"
  full_clone  = true
  scsihw      = "virtio-scsi-pci"

  cores   = 2
  memory  = 4096
  agent   = 0

  disk {
    slot    = 0      # <--- Deve essere una stringa completa
    size    = "20G"
    type    = "scsi"       
    storage = "local-lvm"
  }
  network {
    model  = "virtio"
    bridge = "vmbr0" # Verifica che il tuo bridge si chiami così (standard)
  }
  os_type   = "cloud-init"
  ipconfig0 = "ip=192.168.1.93/24,gw=192.168.1.1"

  ciuser     = "enrico"
  sshkeys    = <<EOF
  ssh-ed25519**********chiave-ssh*************
  EOF
}
```
**testiamo main.tf**
```bash
terraform plan    # premi yes
```


**diamo vita alle due vm**
```bash
terraform apply
```
Appena lanci il primo apply, Terraform crea un file chiamato terraform.tfstate.

Nota bene: Quel file è la "memoria" di Terraform. Se domani cambi il numero di core nel main.tf e rilanci apply, Terraform legge lo stato, vede che la VM esiste già e invece di ricrearla, ne modifica solo i core.

Un piccolo avvertimento
L'unica cosa che Terraform non fa è configurare quello che c'è dentro la VM (installare database, app, o configurare file di sistema complessi) dopo che è stata accesa. Per quello useremo Ansible.

Se da problemi è utile usare questo comando per vedere i log
```bash
TF_LOG=INFO terraform apply
```

“I encountered and mitigated instability issues in the Proxmox Terraform provider by tuning timeouts and switching to API token authentication.”

🧱 1. Proxmox + Terraform: creazione VM automatizzata

Obiettivo
Clonare automaticamente una VM template (ID 100) in più VM tramite Terraform.

Cosa hai fatto

Installato e configurato Terraform

Usato il provider telmate/proxmox

Scritto configurazione per clonare la VM 100 in:

VM 101

VM 102

Problemi incontrati

Errori di permessi (VM.Monitor)

Errori di autenticazione (401 authentication failure)

Provider che crashava dopo l’apply

Soluzione

Capito la differenza tra utenti @pam e @pve

Creato utente Proxmox nativo enrico@pve

Assegnato ruolo PVEAdmin

Aggiornato provider.tf per usare enrico@pve

Le VM vengono create correttamente (anche se il provider resta instabile)

👉 Risultato: infrastruttura VM creata via IaC ✔️

🔐 2. SSH: accesso sicuro e pronto per automazione

Obiettivo
Accedere alle VM clonate senza password, in modo automatizzabile.

Cosa hai fatto

Generato / usato chiavi SSH

Sistemato known_hosts quando le VM clonate avevano chiavi diverse

Effettuato accesso da WSL via SSH key-based

Problema risolto

Errore “REMOTE HOST IDENTIFICATION HAS CHANGED”

Soluzione

Usato ssh-keygen -R <ip>

Riconnesso e accettato la nuova chiave

👉 Risultato: accesso SSH stabile e automatizzabile ✔️

🧠 3. Comprensione architetturale (valore reale)

Hai capito e applicato concetti non banali:

Differenza tra:

utente di sistema (@pam)

utente API (@pve)

Perché l’automazione deve usare utenti Proxmox nativi

Limiti reali del provider Terraform per Proxmox

Perché SSH con chiavi è fondamentale per CI/CD

👉 Questo non è tutorial copying, è troubleshooting vero.

🎓 COME LO PUOI SCRIVERE (CV / README)

Puoi letteralmente scrivere:

Implemented infrastructure automation on Proxmox using Terraform to clone and provision virtual machines from templates. Solved authentication and permission issues by migrating from system users to Proxmox native users and enabling SSH key-based access for automated provisioning.

Oppure più semplice:

Automated VM provisioning on Proxmox using Terraform and prepared the environment for CI/CD automation with SSH key-based access.

LA CHIAVE SSH L'AVEVO MESSA IN CLOUD INIT DELLA VM 100 DA CLONARE E LE ALTRE LA EREDITANO PERMETTENDOMI DI ACCEDERE SENZA PASSWORD

COME FUNZIONANO LE CHIAVI SSH (VERITÀ FONDAMENTALE)

La chiave non è “del PC”, è:

del client SSH


E tu hai due client diversi:

Ambiente	Percorso chiavi
WSL (Linux)	/home/enrico/.ssh/id_ed25519
Windows CMD / PowerShell	C:\Users\Enrico\.ssh\id_ed25519

Sono file diversi.

COSA HAI FATTO (SENZA RENDERTENE CONTO)

Hai messo UNA SOLA chiave pubblica nella VM, probabilmente:
quella generata in WSL

quindi:

WSL entra ✔️
CMD Windows ❌ (usa un’altra chiave)

Non c’è contraddizione.

SOLUZIONE CORRETTA (PROFESSIONALE)
Puoi mettere PIÙ chiavi nello stesso utente

authorized_keys supporta infinite chiavi.

OPZIONE 1 – COPIA LA CHIAVE WINDOWS SULLA VM
1. Vedi la chiave pubblica Windows (CMD o PowerShell):
type C:\Users\Enrico\.ssh\id_ed25519.pub


(se non esiste → ssh-keygen)

2. Sulla VM:
nano ~/.ssh/authorized_keys


Incolla sotto la chiave WSL.

Salva.

Ora:

WSL ✔️
CMD ✔️

OPZIONE 2 – RIUSA LA STESSA CHIAVE (consigliata)

Puoi dire a Windows di usare la chiave WSL.

In PowerShell:

ssh -i \\wsl$\Ubuntu\home\enrico\.ssh\id_ed25519 enrico@192.168.1.101


Oppure copia la chiave WSL in:

C:\Users\Enrico\.ssh\

## disabilita accesso con password in ssh

sudo nano /etc/ssh/sshd_config



