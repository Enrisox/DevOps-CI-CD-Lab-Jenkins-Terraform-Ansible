# Monitoring a VM with Prometheus and Grafana

- VM 101 (Jenkins): Sarà il nostro "Centro di Comando". Qui faremo girare Prometheus (il database dei dati) e Grafana (i grafici).
- VM 102 (Runtime): Sarà la "Sorgente". Qui installeremo un container chiamato node-exporter che monitora CPU, RAM e Rete.

## Installare il "Sensore" sulla VM runtime via Ansible 

sul mio PC Host , da ubuntu terminal che è il mio Ansible Control Node

1. Creo un nuovo playbook:  playbooks/monitoring.yml

```yaml
- name: Install Monitoring Sensors
  hosts: runtime_group
  become: true
  tasks:
    - name: Run Node Exporter Container
      community.docker.docker_container:
        name: node-exporter
        image: prom/node-exporter:latest
        state: started
        restart_policy: always
        published_ports:
          - "9100:9100"
        network_mode: host

```

2. lancio il comando standard per installare il sensore sulla VM di Runtime (102)

```yaml
ansible-playbook -i inventory/hosts.ini playbooks/monitoring.yml
```

3. http://192.168.1.8:9100/metrics

## Installo Prometheus + Grafana sulla VM CI (101)
Ora dobbiamo fare in modo che la VM 101 vada a leggere questi dati dalla VM 102 e ce li mostri in un grafico.

Procediamo creando un playbook che configurerà tutto sulla VM 101.

```yaml
nano ~/lab-ansible/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s # Frequenza di campionamento

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.1.8:9100']
```

## Creo il Playbook playbooks/setup_monitoring_server.yml
Questo playbook farà tre cose sulla VM Jenkins 101:

- Creerà una cartella per i dati.
- Caricherà il file prometheus.yml appena creato.
- Avvierà i container di Prometheus e Grafana.


```yaml
nano playbooks/setup_monitoring_server.yml
```


```yaml
---
- name: Setup Monitoring Server (Prometheus & Grafana)
  hosts: jenkins
  become: true
  tasks:
    - name: Create Prometheus config directory
      ansible.builtin.file:
        path: /etc/prometheus
        state: directory
        mode: '0755'

    - name: Copy Prometheus configuration
      ansible.builtin.copy:
        src: ../prometheus.yml
        dest: /etc/prometheus/prometheus.yml
        mode: '0644'

    - name: Run Prometheus Container
      community.docker.docker_container:
        name: prometheus
        image: prom/prometheus:latest
        state: started
        restart_policy: always
        volumes:
          - /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
        published_ports:
          - "9090:9090"

    - name: Run Grafana Container
      community.docker.docker_container:
        name: grafana
        image: grafana/grafana:latest
        state: started
        restart_policy: always
        published_ports:
          - "3000:3000"
```

3. Eseguo il Playbook **setup_monitoring_server.yml**
```yaml
ansible-playbook -i inventory/hosts.ini playbooks/setup_monitoring_server.yml
```

## Primo Accesso a Grafana
Apri il browser e vai su: http://192.168.1.7:3000

Login:

User: admin
Password: admin

Ti chiederà di cambiare password:

## Collegare Prometheus a Grafana

1. Nella colonna di sinistra, clicca sull'icona dell'ingranaggio o cerca "Connections" -> "Data Sources".
2. Clicca su "Add data source".
3. Seleziona Prometheus.
4. Nel campo URL, scrivi l'indirizzo della VM dove gira Prometheus (la stessa su cui sei ora): http://192.168.1.7:9090
5. Scorri in fondo e clicca su "Save & Test".
6. Deve apparire un messaggio verde: "Data source is working".


**Esistono dashboard già fatte dalla community. La più famosa per il Node Exporter è la 1860.**

- Nel menu data source, in quella appena aggiunta.
- Clicca su "New" e poi su "Import".
- Nel campo "Import via grafana.com", scrivi  1860 e clicca su "Load".  ( 1860 è una dashboard pre-creata e famosa per le sue qualità di monitoraggio server Linux e node exporter( Node Exporter è un “sensore” che gira su un server e legge lo stato del sistema
(CPU, RAM, disco, rete…) e lo rende leggibile a Prometheus.
- Dagli un nome (es. "Monitoraggio VM Proxmox").
- Clicca su "Import".



