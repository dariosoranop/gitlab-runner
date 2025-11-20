# GitLab Runner – Guida alla Configurazione
Questa guida descrive come installare, configurare e registrare GitLab Runner su un nuovo server dedicato, compatibile con GitLab Community Edition (CE) e GitLab Enterprise Edition (EE).  
È pensata per consentire al cliente di configurare autonomamente l’infrastruttura runner.

---

## 1. Requisiti

### Server dedicato per GitLab Runner
- OS consigliato: **Ubuntu 22.04 / Debian 12 / RHEL 8 / Rocky 8 / AlmaLinux 8**
- CPU: 2+ core  
- RAM: 4+ GB  
- Storage: 20+ GB  
- Accesso HTTPS verso:
  - GitLab CE 
  - GitLab EE audit

### Certificati
- **È necessario importare la CA del GitLab CE** per permettere al runner di validare TLS.
- **NON è necessario**:
  - certificato client
  - SAN specifica per la VM
  - certificato firmato per il runner

Il runner deve semplicemente *fidarsi* della CA che firma il certificato del GitLab CE.

---

##  2. Installazione di GitLab Runner

### Per Debian/Ubuntu
```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner
```
### Per RHEL / Rocky / AlmaLinux
```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
sudo yum install gitlab-runner
```
##  3. Import della CA Mediocredito

### Copiare il file della CA in:
```bash
/usr/local/share/ca-certificates/mcc-ca.crt
```
### Aggiornare il sistema:
```bash
sudo update-ca-certificates
```

## Import CA anche in Docker (obbligatorio)
### Creare la cartella:
```bash
sudo mkdir -p /etc/gitlab-runner/certs
sudo cp mcc-ca.crt /etc/gitlab-runner/certs/
```
### Impostare i permessi:
```bash
sudo chown gitlab-runner:gitlab-runner /etc/gitlab-runner/certs/*
```
##  4. Registrazione dello Shared Runner (UI)
1. Accedere al GitLab EE Audit
2. Vai su:
```bash
Admin → CI/CD → Runners → New runner
```
3. Seleziona:
   docker executor
   immagine base: alpine:latest o python:3.12

4. Copia il comando fornito dalla UI per registrare il runner:
```bash
   sudo gitlab-runner register
```
5. Quando richiesto:
   - URL GitLab EE → https://<gitlab-ee>
   - Token → fornito dalla UI
   - Description → shared-runner
   - Tags → shared, security
   - Executor → docker
   - Docker image → alpine:latest

Al termine:
```bash
sudo systemctl restart gitlab-runner
```
##  Registrazione Project Runner (UI)
Per ogni progetto:
1. Apri il progetto su GitLab EE:
```bash
   Project → Settings → CI/CD → Runners
```
2. Sezione "Set up a runner manually"
3. Copia il comando di registrazione e lancia:
```bash
sudo gitlab-runner register
```
4. Imposta:
   Runner non shared
   - Tags → ad esempio:
     - frontend, sast, dast
     - backend, sast, ds
### Consigliato:
Ogni progetto dovrebbe avere almeno un runner dedicato, ottimizzato per SAST/DAST.

##  Verifica del Runner
```bash
sudo gitlab-runner verify
sudo gitlab-runner list
```
## Nella UI dovresti vedere:
   - Runner active
   - green heartbeat
   - Last contact < 1 min

## Configurazione runner config.toml
```bash
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "runner-pull-mirror"
  url = "https://gitlab-ultimate.mcc.it"
  id = 1
  token = "glrt-Lu-fYX-vEBxzyfqwcbqrE286MQp0OjEKdToxCw.01.121eyku3p"
  executor = "docker"

  [runners.cache]
    MaxUploadedArchiveSize = 0

  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false

    volumes = [
      "/etc/gitlab-runner/certs:/etc/gitlab-runner/certs:ro",
      "/var/run/docker.sock:/var/run/docker.sock",
      "/cache"
    ]

    shm_size = 0
```
### Riferimenti ufficiali
- https://docs.gitlab.com/runner/install/
- https://docs.gitlab.com/runner/configuration/advanced-configuration/
- https://docs.gitlab.com/runner/configuration/tls-self-signed/
- https://docs.gitlab.com/ci/runners/configure_runners/#registering-a-project-runner
- https://docs.gitlab.com/ci/runners/configure_runners/#registering-a-shared-runner
