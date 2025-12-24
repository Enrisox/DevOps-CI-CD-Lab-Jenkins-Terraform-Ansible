# Installazione Proxmox 

1) installato proxmox da chiavetta usb contenete ISO , F1 per entrare nel Bios e impostare chiavetta usb come metodo di boot principale.
2) installare graphical mode, scegli ip statico e hostname e password di root..
3) apt update e apt upgrade : darà errore per colpa della licenza mancante quindi bisogna cancellare o commentare contenuto sia di apt/sources.list.d/ceph.sources che di  /etc/apt/sources.list.d/pve-enterprise.sources
4) nano /etc/apt/sources.list.d/pve-no-subscription.sources   e incolla dentro
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Architectures: amd64


5)aggiungiamo nuovo utente sul server con comando linux adduser e aggiungiamo utente con stesso nome e password anche sulla gui di proxmox , raggiungibile a ip:8006
6)creo vm ubuntu server con 1 core e 2 gb di ram.. 
7)finita istallazione procedo a confogirazione rete.. 
 
