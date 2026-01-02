# CONFIGURAZIONE E INSTALLAZIONE JENKINS SU VM CI (continuous integration)

**Jenkins** è un server di automazione (tipicamente self‑hosted) usato soprattutto per CI/CD: definisci una “pipeline” (script) che esegue build, test, scan, packaging e deploy quando succede un evento (push, PR, cron, manuale) o quando lo decidi tu.

**A cosa serve Jenkins**
- Orchestrare step di pipeline (build/test/deploy) su uno o più agent/nodi, anche in parallelo.
- Integrare tanti tool diversi grazie ai plugin (Git, Docker, Kubernetes, scanner sicurezza, notifiche, ecc.).
- Tenere tutto “in casa” (homelab/on‑prem) e avere controllo totale su rete, credenziali, runner, caching e ambienti.

**Differenza rispetto a GitHub Actions**
GitHub Actions è integrato in GitHub (SaaS) con runner gestiti o self-hosted; Jenkins di solito lo gestisci tu su VM/container.
**Setup/gestione**: Actions è più “pronto subito”; Jenkins richiede più manutenzione (aggiornamenti, plugin, backup, sicurezza), ma è più flessibile quando vuoi ambienti custom.
**Integrazione repo**: Actions è nativo per GitHub; Jenkins è agnostico (GitHub/GitLab/Bitbucket/etc.), ma l’integrazione la configuri tu.

**Differenza rispetto ai servizi AWS simili (CodeBuild/CodePipeline)**
- **Modello**: su AWS hai servizi gestiti e paghi a consumo; Jenkins è un tuo servizio (costi “fissi” di VM e tempo di gestione).
- **Integrazione cloud**: AWS CI/CD è fortemente integrato con IAM, ECR, ECS/EKS, S3, ecc.; Jenkins può farlo, ma via plugin/credential management e configurazione.
- **Portabilità**: Jenkins ti resta uguale in homelab, on‑prem o cloud; con AWS sei più “cloud‑native” ma anche più legato all’ecosistema.

# Creazione del Playbook Jenkins


configureremo un Volume Persistente. Questo significa che se cancelli il container o riavvii la VM, i tuoi lavori (Jobs), i plugin e le configurazioni di Jenkins non andranno persi perché saranno salvati in una cartella sulla VM (/home/enrico/jenkins_home).

Procediamo a scrivere il file playbooks/deploy-jenkins.yml
```bash
nano playbooks/deploy-jenkins.yml
```

```bash
---
- name: Deploy Jenkins via Docker
  hosts: jenkins
  become: true

  tasks:
    - name: Assicura che la directory dei dati esista
      ansible.builtin.file:
        path: /home/enrico/jenkins_home
        state: directory
        owner: 1000 # UID standard dell'utente jenkins nel container
        group: 1000
        mode: '0755'

    - name: Avvia il container di Jenkins
      community.docker.docker_container:
        name: jenkins-server
        image: jenkins/jenkins:lts
        state: started
        restart_policy: always
        published_ports:
          - "8080:8080"   # Interfaccia Web
          - "50000:50000" # Porta per i futuri agent (nodi worker)
        volumes:
          - /home/enrico/jenkins_home:/var/jenkins_home
        env:
          JAVA_OPTS: "-Djenkins.install.runSetupWizard=true"
```

**Cos’è ansible-galaxy collection install community.docker**
Ansible Galaxy è il repository ufficiale di ruoli e collection Ansible.
Una collection è un pacchetto che contiene moduli, plugin e ruoli specifici per un certo dominio.
community.docker è la collection ufficiale della community per gestire Docker con Ansible.
Ansible ha bisogno di un "modulo" specifico per parlare con Docker. Se non lo hai già, installalo nel WSL con questo comando:

```bash
ansible-galaxy collection install community.docker

ansible-playbook -i inventory/hosts.ini playbooks/deploy-jenkins.yml
```

Scarica i moduli più recenti per Docker, container, immagini, network, volume, compose, ecc.
Li mette in ~/.ansible/collections o nella cartella collections/ del progetto.

## Una volta che il playbook dice "OK", Jenkins sarà attivo. Ecco come procedere:

Apri il browser e vai su: http://192.168.1.7:8080
Ti apparirà una schermata che chiede la Administrator Password.
Per recuperarla senza entrare nella VM, usa Ansible dal tuo WSL


**È un file temporaneo che Jenkins crea al primo avvio per assicurarsi che solo chi ha accesso al server possa configurarlo.**
```bash
ansible ci -a "cat /home/enrico/jenkins_home/secrets/initialAdminPassword"
```
**installa componenti aggiuntivi --> OK**




# Dobbiamo configurare il "Nodo Agente". In pratica:

- Creeremo una chiave SSH per permettere a Jenkins di entrare nella macchina .8.
- Configureremo la macchina .8 dentro l'interfaccia di Jenkins.
- Faremo un test: chiederemo a Jenkins di lanciare un comando sulla .8 per vedere se "ubbidisce".

Ora faremo in modo che il "Capocantiere" (Jenkins sulla .7) possa dare ordini all' "Operaio" (Runtime sulla .8). Per farlo, useremo una chiave SSH.

Il trucco è questo: genereremo la chiave sulla macchina .7 (dentro la cartella di Jenkins) e la copieremo sulla macchina .8.

1. Genera la chiave SSH sul server Jenkins
Lancia questo comando dal tuo WSL. Useremo Ansible per dire alla macchina .7 di creare una chiave proprio nella cartella che abbiamo mappato per Jenkins:


```bash
ansible ci -m shell -a "ssh-keygen -t ed25519 -f /home/enrico/jenkins_home/jenkins_key -N ''"
```
2. Autorizza Jenkins sulla macchina Runtime
Ora dobbiamo prendere la "chiave pubblica" appena creata e metterla tra le chiavi autorizzate della macchina .8. Invece di fare copia-incolla manuale, facciamo tutto con un colpo solo:

```bash
# 1. Leggiamo la chiave pubblica dalla .7
PUBLIC_KEY=$(ansible ci -a "cat /home/enrico/jenkins_home/jenkins_key.pub" | grep ssh-ed25519)

# 2. La aggiungiamo alla .8 (Runtime)
ansible runtime -m authorized_key -a "user=enrico key='$PUBLIC_KEY'"
```

## sulla macchina runtime

Sul terminale della Vm runtime ho installato Java con questi comandi:


```bash
sudo apt update                      #aggiorna i pacchetti
sudo apt install default-jre -y        #Installa Java (JRE)
java -version                     #verifica l'installazione:
```

## creazione nodo da interfaccia jenkins http://ip_VM_Jenkins(CI)


1. Vai su Gestisci Jenkins (Manage Jenkins) > Nodes.
2. Clicca su + New Node (Nuovo Nodo) a sinistra.
3. Inserisci il nome runtime-node, seleziona Permanent Agent (Agente permanente) e clicca su Create.
4. Remote root directory: Scrivi /home/enrico/jenkins (la cartella dove Jenkins lavorerà sul nodo).
5. Labels: Scrivi runtime (serve per dire ai futuri lavori/job di girare proprio qui).
6. Launch method: Seleziona Launch agents via SSH.
7. Host: IP statico della VM runtime: es: 192.168.1.8.

**Credentials:**

1. Clicca su Add > Jenkins.
2. Kind: Seleziona SSH Username with private key.
3. Username: enrico.
4. Private Key: Seleziona Enter directly, clicca su Add e incollato la chiave privata che leggo dal terminale con il comando ansible.
5. Clicca su Add in fondo al pop-up.
6. seleziona la credenziale appena creata dal menu a tendina Credentials.

```bash
ansible ci -a "cat /home/enrico/jenkins_home/jenkins_key"
```

Host Key Verification Strategy: Cambia in Non verifying Verification Strategy (per saltare il controllo manuale dell'impronta SSH).

**Save**

Se nei log del nodo c'è **"Agent successfully connected and online"** è funzionante.

