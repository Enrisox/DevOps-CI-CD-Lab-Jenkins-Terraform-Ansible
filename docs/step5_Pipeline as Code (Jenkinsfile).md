# Pipeline as Code (Jenkinsfile)

1. Da dashboard Jenkins --> nuovo elemento
2. creiamo una Pipeline. Questo ci dirà se il nodo runtime è davvero pronto per il lavoro serio.
3. Dalla Dashboard di Jenkins, clicca su Nuovo elemento (New Item).
4. Nome: Verifica-Ambiente-Runtime.
5. Seleziona Pipeline e clicca su OK.
6. Scorri fino in fondo alla sezione Pipeline e incolla questo codice:
7. Clicca su compila ora a SX.

**Prima pipeline di prova**

```bash
pipeline {
    agent { label 'runtime' } // Forza l'esecuzione sul tuo nodo .8

    stages {
        stage('Verifica Requisiti') {
            steps {
                echo 'Verifico la versione di Java...'
                sh 'java -version'
                
                echo 'Verifico se Docker è installato e funzionante...'
                sh 'docker version'
                
                echo 'Verifico i permessi dell utente...'
                sh 'groups'
            }
        }
    }
}
```

- Java: Se Jenkins è online, sappiamo che c'è, ma questo conferma la versione.
- Docker Version: Se questo comando fallisce, il tuo script di bootstrap non ha installato Docker.
- Groups: Se nell'output non vedi la parola docker, Jenkins non avrà i permessi per lanciare container e dovrai aggiungerlo al gruppo.

## Pipeline di Continuous Deployment (CD)
Visto che il nodo è pronto, creiamo una pipeline che simula un vero deploy. Questa pipeline pulirà eventuali vecchi container e lancerà un'applicazione web.

**installare AWS CLI v2 sul nodo runtime**

Per scompattare l'installatore di AWS, serve **unzip**. Eseguilo sul terminale del nodo:

```bash
sudo apt update && sudo apt install unzip curl -y
```
## Scarica e installa AWS CLI v2
Copia e incolla questi tre comandi uno alla volta:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"  # scarica il pacchetto ufficiale
unzip awscliv2.zip        #Estrai i file
sudo ./aws/install        #Esegui l'installazione
```
**Verifica l'installazione**
```bash
aws --version
```
Dovresti vedere qualcosa come aws-cli/2.x.x. Se lo vedi, il nodo è finalmente pronto per parlare con ECR.

- Login ad ECR: La pipeline userà il comando aws ecr get-login-password per autenticare Docker verso AWS.
- Pull dell'immagine: Docker scaricherà la tua immagine specifica dalla tua repo ECR.
- Run del container: L'app verrà lanciata sul nodo runtime

## creazione user nel mio aws per ottenere le chiavi

**Installa il Plugin AWS**

1. Dalla dashboard principale di Jenkins, clicca su rotellina
2. Clicca su Plugins (o Manage Plugins).
3. Vai nella scheda Available plugins (Plugin disponibili) in alto.
4. Nella barra di ricerca a destra, scrivi: AWS Credentials.
5. Spunta la casella del plugin AWS Credentials (solitamente l'autore è CloudBees o Jenkins Project).
6. Clicca sul tasto Install


**Bisogna generare una coppia di chiavi: Access Key ID e Secret Access Key.**

1. Accedi alla Console AWS e cerca il servizio IAM.
2. Nel menu a sinistra, clicca su Users (Utenti) e poi su Create user.
3. Assegna un nome (es: jenkins-runtime-user). Non selezionare l'accesso alla console web; questo utente serve solo per l'accesso programmatico (API).
4. Nella schermata dei permessi (Set permissions), seleziona Attach policies directly.
5. Cerca e seleziona la policy: AmazonEC2ContainerRegistryReadOnly. Questa policy permette a Jenkins di vedere le immagini e scaricarle (pull), ma non di cancellarle o caricarne di nuove, aumentando la sicurezza.
6. Completa la creazione dell'utente.
7. Una volta creato l'utente, clicca sul suo nome, vai nella scheda Security credentials e clicca su Create access key.
8. Seleziona il caso d'uso Command Line Interface (CLI) e vai avanti fino a ottenere i due codici.

## salviamo credentiazli iam user appena creato, su jenkins credentials globali

1. impostazioni --> credentials --> global --> aggiungi una credential
2. Type: AWS Credentials
3. ID: aws-ecr-creds.
4. Description: Una descrizione a tua scelta (es. "Chiavi per AWS ECR").
5. Access Key ID: Incolla il codice che inizia con AKIA... che hai preso da AWS IAM.
6. Secret Access Key: Incolla la tua chiave segreta lunga.
7. Salva credential

## Creo una pipeline che pulla un' image dalla mia repo su ECR in AWS 


```bash
pipeline {
    agent { label 'runtime' } // Assicurati che il tuo nodo "runtime" sia online

    environment {
        // --- DATI REALI DAL TUO ECR ---
        AWS_ACCOUNT_ID = '***********'
        AWS_REGION     = 'eu-west-1'
        REPO_NAME      = 'quiz-app'
        // ------------------------------
        
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {
        stage('ECR Login') {
            steps {
                echo "Autenticazione su AWS ECR in corso ..."
                // Usa le credenziali 'aws-ecr-creds' che hai salvato prima
                withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds' ]]) {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}"
                }
            }
        }

        stage('Pull Image') {
            steps {
                echo "Download dell'immagine ${REPO_NAME}:latest..."
                sh "docker pull ${ECR_URL}/${REPO_NAME}:latest"
            }
        }

        stage('Deploy Container') {
            steps {
                echo "Avvio dell'applicazione sul nodo runtime..."
                // Pulizia vecchi container per evitare conflitti
                sh "docker rm -f quiz-app-container || true"
                
                // Lancio del container sulla porta 8080 
                sh "docker run -d --name quiz-app-container -p 8080:5000 ${ECR_URL}/${REPO_NAME}:latest"
            }
        }
    }

    post {
        success {
            echo "Deploy riuscito! L'app è disponibile su http://IP-VM-RUNTIME:8080"
        }
        failure {
            echo "Il deploy è fallito. Controlla il 'Console Output' per i dettagli."
        }
    }
}
```

### sicurezza

- Segretezza delle Credenziali: Le tue chiavi AWS (AKIA...) non appaiono mai nel log di Jenkins né nello script. Sono protette dal plugin Credentials che le "maschera" (vedresti solo **** nei log).
- Principio del Minimo Privilegio: L'utente IAM che abbiamo creato ha solo il permesso ReadOnly. Anche se qualcuno rubasse quelle chiavi, potrebbe solo scaricare le immagini, ma non potrebbe cancellare i tuoi database o creare nuove macchine costose su AWS.
- Autenticazione Effimera: Usando get-login-password, non salviamo la password di Docker sul disco in modo permanente. Il login è valido solo per quella specifica sessione di lavoro.

1)**BLOCCO AGENT**

**agent { label 'runtime' }** in una Jenkins Pipeline significa: “Questo stage (o l’intera pipeline) deve girare su un agent Jenkins che ha l’etichetta runtime”, cioè su una macchina registrata come worker (nel tuo caso la VM 192.168.1.8), non sul controller. In Jenkins il controller orchestra e schedula, mentre l’agent esegue davvero comandi, build, docker, ecc.

**Modello “SSH da controller” vs “agent runtime”**

**Modello SSH (anti-pattern tipico)**
Il controller esegue uno step tipo ssh runtime "docker pull ... && docker compose up -d".

Quindi il controller deve avere:

chiavi SSH/credenziali per entrare in runtime,
rete aperta verso runtime (SSH in ingresso sul runtime, firewall più permissivo),
logica di deploy “remota” fragile (timeout SSH, chiavi, jump host, ecc.).
Problema tecnico principale: stai trasformando il controller in una macchina “che entra nei server”

**Modello agent**
La VM runtime esegue un processo “agent” Jenkins.

- L’agent si connette al controller e resta in ascolto di job (concetto: pull model / connessione uscente).
- Quando nel Jenkinsfile metti agent { label 'runtime' }, Jenkins:
- trova un agent online con quell’etichetta e prepara workspace lì, invia lo script dei passaggi,
- i comandi (es. docker pull, docker compose up) girano localmente sulla VM runtime.

**Separazione netta dei ruoli (control plane vs data plane)**
- **Controller**: decisioni, scheduling, UI, credenziali centralizzate, audit pipeline.
- **Runtime agent**: esecuzione dei comandi e accesso alle risorse locali (socket Docker, filesystem, rete).

### Scalabilità e ripetibilità
Con label:

- aggiungi altri runtime node (es. runtime-1, runtime-2) con la stessa label e Jenkins bilancia.
- se domani passi da una VM a tre VM o a un cluster, l’idea rimane identica: il Jenkinsfile non deve diventare un groviglio di ssh

**Sposto l’esecuzione sul nodo target via agent, uso connessioni uscenti e riduco la necessità di credenziali di accesso remoto; il controllo resta nel sistema CI con audit e permessi.**

2)**BLOCCO ENVIROMENT**
```bash

environment {
    AWS_ACCOUNT_ID = '266735824805'
    AWS_REGION     = 'eu-west-1'
    REPO_NAME      = 'quiz-app'
    ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
}
```

**Definisce variabili riutilizzabili:**

- Account AWS
- Regione (Irlanda)
- Nome repo ECR
- URL completo del registry

Evita hardcoding nei comandi
Pipeline più leggibile e manutenibile

**3)STAGE ECR LOGIN**
```bash

Stage: ECR Login
stage('ECR Login') {
    withCredentials([...]) {
        sh "aws ecr get-login-password | docker login ..."
    }
}
```

1. Usa credenziali AWS salvate in Jenkins (aws-ecr-creds)
2. Recupera un token temporaneo da ECR
3. Fa login Docker verso AWS ECR

👉 Non salvi password in chiaro
👉 Sicurezza corretta
👉 Best practice AWS

**4)STAGE PULL IMAGE**
```bash
docker pull 2667...amazonaws.com/quiz-app:latest
```
Scarica l’ultima versione dell’immagine Docker dal registry ECR.

**5)Stage: Deploy Container**

```bash
docker rm -f quiz-app-container || true
```
Ferma e rimuove il container esistente
👉 Deploy idempotente


