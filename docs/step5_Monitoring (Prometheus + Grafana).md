# Monitoring

VM 101 (Jenkins): Sarà il nostro "Centro di Comando". Qui faremo girare Prometheus (il database dei dati) e Grafana (i grafici).

VM 102 (Runtime): Sarà la "Sorgente". Qui installeremo un container chiamato node-exporter che monitora CPU, RAM e Rete.

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
