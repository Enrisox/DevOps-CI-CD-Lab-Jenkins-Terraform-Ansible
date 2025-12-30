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
 
8)creare è un Cloud-Init drive. Nel mondo dell'automazione (Terraform/Ansible), è il "pezzo del puzzle" fondamentale.

Senza un drive Cloud-Init, quando Terraform crea una VM su Proxmox, la VM si accenderebbe ma rimarrebbe ferma alla schermata di installazione o di login, obbligandoti a intervenire a mano. Con il drive Cloud-Init, Terraform può "iniettare" automaticamente l'utente, la password, le chiavi SSH e l'indirizzo IP.


Nelle specifiche Proxmox per i template:

- IDE 2: È lo standard de facto per montare l'immagine ISO virtuale che contiene i dati di configurazione (user-data). Viene visto dalla VM come un CD-ROM da cui leggere le istruzioni al primo avvio.
- Storage Local: È dove viene salvato questo piccolo file di configurazione.

9)da powershell su windows installo sottosistema linux WLS. 
wsl --install

10)genera chiavi ssh pc casa e mostrala 
ssh-keygen -t ed25519 -C "PC-CASA"
cat ~/.ssh/id_ed25519.pub

11)copiala tutta, anche commento


