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



