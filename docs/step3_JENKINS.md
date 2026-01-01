# CONFIGURAZIONE E INSTALLAZIONE JENKINS SU VM CI (continuous integration)

Jenkins è un server di automazione (tipicamente self‑hosted) usato soprattutto per CI/CD: definisci una “pipeline” (script) che esegue build, test, scan, packaging e deploy quando succede un evento (push, PR, cron, manuale) o quando lo decidi tu.

A cosa serve Jenkins (in breve)
Orchestrare step di pipeline (build/test/deploy) su uno o più agent/nodi, anche in parallelo.

Integrare tanti tool diversi grazie ai plugin (Git, Docker, Kubernetes, scanner sicurezza, notifiche, ecc.).

Tenere tutto “in casa” (homelab/on‑prem) e avere controllo totale su rete, credenziali, runner, caching e ambienti.

Differenza rispetto a GitHub Actions
Dove gira: GitHub Actions è integrato in GitHub (SaaS) con runner gestiti o self-hosted; Jenkins di solito lo gestisci tu su VM/container.

Setup/gestione: Actions è più “pronto subito”; Jenkins richiede più manutenzione (aggiornamenti, plugin, backup, sicurezza), ma è più flessibile quando vuoi ambienti custom.

Integrazione repo: Actions è nativo per GitHub; Jenkins è agnostico (GitHub/GitLab/Bitbucket/etc.), ma l’integrazione la configuri tu.

Differenza rispetto ai servizi AWS simili (CodeBuild/CodePipeline)
Modello: su AWS hai servizi gestiti e paghi a consumo; Jenkins è un tuo servizio (costi “fissi” di VM e tempo di gestione).

Integrazione cloud: AWS CI/CD è fortemente integrato con IAM, ECR, ECS/EKS, S3, ecc.; Jenkins può farlo, ma via plugin/credential management e configurazione.

Portabilità: Jenkins ti resta uguale in homelab, on‑prem o cloud; con AWS sei più “cloud‑native” ma anche più legato all’ecosistema.

configureremo un Volume Persistente. Questo significa che se cancelli il container o riavvii la VM, i tuoi lavori (Jobs), i plugin e le configurazioni di Jenkins non andranno persi perché saranno salvati in una cartella sulla VM (/home/enrico/jenkins_home).

Procediamo a scrivere il file playbooks/deploy-jenkins.yml
