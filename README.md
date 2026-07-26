[EN COURS]

# Homelab



> Infrastructure personnelle auto-hébergée sur deux machines physiques — un HP EliteDesk 800 G3 Mini sous Proxmox VE pilotant une VM Docker avec tous les services, et un Acer Gateway DT55 sous Debian servant de NAS avec 2,5 To de stockage.

---

## Sommaire

- [Architecture](#architecture)
- [HP EliteDesk 800 G3 Mini — Proxmox + Docker](#hp-elitedesk-800-g3-mini--proxmox--docker)
  - [Proxmox VE — Création de la VM Docker](#proxmox-ve--création-de-la-vm-docker)
  - [Installation de Docker](#installation-de-docker)
  - [Portainer](#portainer)
  - [Nextcloud](#nextcloud)
  - [Nginx Proxy Manager](#nginx-proxy-manager)
  - [Nginx](#nginx)
  - [Uptime Kuma](#uptime-kuma)
  - [Ollama + Open WebUI](#ollama--open-webui)
  - [HomePage](#homepage)
  - [Pi-Hole](#pi-hole)
- [Acer Gateway DT55 — NAS Debian](#acer-gateway-dt55--nas-debian)
  - [Samba](#samba)

---

## Architecture

Le homelab repose sur deux machines physiques avec des rôles distincts.

![Schéma d'architecture du homelab](VOTRE_SCREENSHOT_SCHEMA_ICI)

| Machine | OS | Rôle |
|---|---|---|
| **HP EliteDesk 800 G3 Mini** | Proxmox VE (Hyperviseur T1) | VM Docker — tous les services |
| **Acer Gateway DT55** | Debian 13 (sans GUI) | NAS — partage de fichiers 2,5 To |
| **HP 17-cn3078nf** | Windows 11 & Ubuntu | PC principal — gestion SSH & navigateur |

---

## HP EliteDesk 800 G3 Mini — Proxmox + Docker

Le HP EliteDesk tourne sous **Proxmox VE**, un hyperviseur de type 1 qui permet de créer et gérer des machines virtuelles directement sur le matériel. Une VM **Debian 13** héberge l'ensemble des services Docker.

<img width="1919" height="949" alt="Capture d&#39;écran 2026-07-25 160735" src="https://github.com/user-attachments/assets/35951f57-a12c-4a5a-9b1b-8f6eb240e278" />

### Proxmox VE — Création de la VM Docker

La VM est créée depuis l'interface web Proxmox avec les paramètres suivants.


**Onglet Général** — nom `VM-Docker`, VM ID `100`, nœud `proxmox`.
<img width="777" height="581" alt="Capture d&#39;écran 2026-07-26 131832" src="https://github.com/user-attachments/assets/d9133d94-588c-43b0-871b-7d0c8141e034" />


**Onglet Système d'exploitation** — image ISO `debian-13.6.0-amd64-netinst`, type Linux.
<img width="777" height="591" alt="Capture d&#39;écran 2026-07-26 131728" src="https://github.com/user-attachments/assets/420e534f-ea8b-48c9-a8bc-2efe57c05084" />


**Onglet Système** — BIOS SeaBIOS, machine i440fx, contrôleur SCSI VirtIO.
<img width="782" height="586" alt="Capture d&#39;écran 2026-07-26 131821" src="https://github.com/user-attachments/assets/6bf93bba-cf40-4174-8cce-8960212b6eee" />


**Onglet Disques** — disque IDE de 30 GiB sur `local-lvm`.
<img width="777" height="588" alt="Capture d&#39;écran 2026-07-26 131918" src="https://github.com/user-attachments/assets/947b7074-ef16-4f00-8557-e3748968ebf1" />


**Onglet Processeur** — 1 socket, 2 cœurs (x86-64-v2-AES).
<img width="774" height="588" alt="Capture d&#39;écran 2026-07-26 132010" src="https://github.com/user-attachments/assets/fb0fecc1-1281-4d9a-91c0-f81b19f2d953" />


**Onglet Mémoire** — 2048 MiB de RAM.
<img width="769" height="588" alt="Capture d&#39;écran 2026-07-26 132046" src="https://github.com/user-attachments/assets/9a1fd114-3211-46d9-ba29-c81a27cdcba4" />


**Onglet Réseau** — pont `vmbr0`, modèle Intel E1000, pare-feu activé.
<img width="774" height="582" alt="Capture d&#39;écran 2026-07-26 132108" src="https://github.com/user-attachments/assets/7c9e8af7-d4a6-49f5-9b2e-86b016146435" />


**Onglet Confirmation** — récapitulatif complet avant création.
<img width="774" height="598" alt="Capture d&#39;écran 2026-07-26 132127" src="https://github.com/user-attachments/assets/4d071259-8fb4-4dea-b8ec-ff8bc91d46c5" />

---

### Installation de Docker

Installation de la version officielle Docker CE sur la VM Debian 13, via le dépôt officiel Docker.

```bash
# Supprimer les anciennes versions si présentes
sudo apt remove docker docker-engine docker.io containerd runc

# Prérequis
sudo apt update
sudo apt install ca-certificates curl gnupg

# Ajout de la clé GPG officielle Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajout du dépôt Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installation
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Vérification
sudo docker run hello-world

# Ajouter l'utilisateur courant au groupe docker (évite sudo)
sudo usermod -aG docker $USER
```

---

### Portainer

**Portainer** est une interface web de gestion des conteneurs Docker. Il permet de voir, démarrer, arrêter et configurer tous les conteneurs sans passer par le terminal.

**Accès :** `https://ip:9443`

```bash
mkdir -p ~/docker/portainer
nano ~/docker/docker-compose.yml
```

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    ports:
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer:/data
```

```bash
cd ~/docker
sudo docker compose up -d

# Récupérer le Setup Token au premier démarrage
sudo docker logs portainer
```

![Portainer](VOTRE_SCREENSHOT_PORTAINER_ICI)

---

### Nextcloud

**Nextcloud** est un cloud personnel auto-hébergé. Il permet de synchroniser et partager fichiers, calendriers et contacts, comme Google Drive mais sur sa propre infrastructure.

**Accès :** `http://ip:8080`

```bash
mkdir -p ~/docker/nextcloud
```

```yaml
# À ajouter dans ~/docker/docker-compose.yml
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    ports:
      - "8080:80"
    volumes:
      - ./nextcloud:/var/www/html
```

```bash
cd ~/docker
sudo docker compose up -d
```

![Nextcloud](VOTRE_SCREENSHOT_NEXTCLOUD_ICI)

---

### Nginx Proxy Manager

**Nginx Proxy Manager** est un reverse proxy avec interface graphique. Il redirige les requêtes web vers les bons conteneurs selon le domaine ou le chemin, et gère les certificats SSL/TLS automatiquement via Let's Encrypt.

**Accès :** `http://ip:81` (interface d'administration)

```bash
mkdir -p ~/docker/nginx
```

```yaml
# À ajouter dans ~/docker/docker-compose.yml
  nginx:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./nginx/data:/data
      - ./nginx/letsencrypt:/etc/letsencrypt
```

```bash
cd ~/docker
sudo docker compose up -d
```

![Nginx Proxy Manager](VOTRE_SCREENSHOT_NPM_ICI)

---

### Nginx

**Nginx** utilisé ici comme serveur web statique pour tester et servir des pages HTML directement depuis la machine.

**Accès :** `http://ip:8081`

```bash
mkdir ~/nginx-web
cd ~/nginx-web
nano index.html
```

```html
<h1>Hello World!</h1>
```

```bash
sudo docker run -d --name mon-serveur-web -p 8081:80 -v ~/nginx-web/index.html:/usr/share/nginx/html/index.html nginx
```

![Nginx](VOTRE_SCREENSHOT_NGINX_ICI)

---

### Uptime Kuma

**Uptime Kuma** est un outil de monitoring de disponibilité. Il vérifie à intervalles réguliers que chaque service du homelab répond correctement, et envoie des alertes en cas de panne.

**Accès :** `http://ip:3001`

```bash
mkdir ~/ai-tools
cd ~/ai-tools
nano docker-compose.yml
```

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    ports:
      - "3001:3001"
    volumes:
      - ./kuma_data:/app/data
```

```bash
sudo docker compose up -d
```

![Uptime Kuma](VOTRE_SCREENSHOT_UPTIMEKUMA_ICI)

---

### Ollama + Open WebUI

**Ollama** est un moteur d'exécution de modèles de langage (LLM) en local. **Open WebUI** est l'interface web qui lui est connectée, offrant une expérience similaire à ChatGPT mais entièrement hébergée sur la machine, sans envoyer de données à des serveurs externes.

**Accès :** `http://ip:3000`

```bash
mkdir ~/ollama-webui && cd ~/ollama-webui
nano docker-compose.yml
```

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - open_webui_data:/app/backend/data
    restart: unless-stopped

volumes:
  ollama_data:
  open_webui_data:
```

```bash
sudo docker compose up -d
```

![Ollama + Open WebUI](VOTRE_SCREENSHOT_OLLAMA_ICI)

---

### HomePage

**HomePage** est un tableau de bord personnalisable qui centralise tous les services du homelab sur une seule page. Il affiche les liens, statuts et informations de chaque service en temps réel.

**Accès :** `http://ip:3005`

```bash
mkdir ~/homepage && cd ~/homepage
nano docker-compose.yml
```

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - "3005:3000"
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=192.168.1.44:3005
    volumes:
      - ./config:/app/config
    restart: unless-stopped
```

```bash
sudo docker compose up -d
```

![HomePage](VOTRE_SCREENSHOT_HOMEPAGE_ICI)

---

### Pi-Hole

**Pi-Hole** est un bloqueur de publicités et de trackers au niveau réseau. Il agit comme un serveur DNS local et filtre les requêtes indésirables pour tous les appareils connectés au réseau domestique, sans configuration nécessaire sur chaque appareil.

**Accès :** `http://ip:VOTRE_PORT/admin`

> 🔧 **Installation à venir**

---

## Acer Gateway DT55 — NAS Debian

L'Acer Gateway DT55 tourne sous **Debian 13 sans interface graphique** pour économiser les ressources (4 Go de RAM). Il sert exclusivement de NAS grâce à ses **2,5 To de stockage** (disque d'origine 500 Go + disque de 2 To ajouté).

---

### Samba

**Samba** permet de partager les 2,5 To de stockage de l'Acer sur le réseau local, rendant les fichiers accessibles depuis n'importe quel appareil (Windows, Linux, macOS) comme un lecteur réseau classique.

![Samba](VOTRE_SCREENSHOT_SAMBA_ICI)

---

## Conclusion

Ce homelab repose sur une séparation claire des rôles : le HP EliteDesk concentre toute la puissance de calcul et l'orchestration Docker via Proxmox, tandis que l'Acer Gateway joue le rôle de NAS dédié au stockage. Cette architecture permet d'héberger des services personnels complets — cloud, IA locale, monitoring, reverse proxy — tout en gardant la main sur ses données.

---

*Réalisé par [Yacine Harrache](https://github.com/yacinehrc) — BTS SIO SLAM | EPSI Lille*
