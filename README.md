**[EN COURS]**

# Homelab

![HomePage — tableau de bord des services](VOTRE_SCREENSHOT_HOMEPAGE_ICI)

> Infrastructure personnelle auto-hébergée sur deux machines physiques — un HP EliteDesk 800 G3 Mini sous Proxmox VE pilotant une VM Docker avec tous les services, et un Acer Gateway DT55 sous Debian servant de NAS avec 2,5 To de stockage via LVM et Samba.

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
  - [Choix du stockage — LVM plutôt que RAID ou NAS](#choix-du-stockage--lvm-plutôt-que-raid-ou-nas)
  - [Création du LVM](#création-du-lvm)
  - [Samba](#samba)
  - [Webmin](#webmin)

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

![Interface Proxmox VE](VOTRE_SCREENSHOT_131220_ICI)

### Proxmox VE — Création de la VM Docker

La VM est créée depuis l'interface web Proxmox avec les paramètres suivants.

**Onglet Général** — nom `VM-Docker`, VM ID `100`, nœud `proxmox`.

![Général](VOTRE_SCREENSHOT_131728_ICI)

**Onglet Système d'exploitation** — image ISO `debian-13.6.0-amd64-netinst`, type Linux.

![Système d'exploitation](VOTRE_SCREENSHOT_131821_ICI)

**Onglet Système** — BIOS SeaBIOS, machine i440fx, contrôleur SCSI VirtIO.

![Système](VOTRE_SCREENSHOT_131832_ICI)

**Onglet Disques** — disque IDE de 30 GiB sur `local-lvm`.

![Disques](VOTRE_SCREENSHOT_131918_ICI)

**Onglet Processeur** — 1 socket, 2 cœurs (x86-64-v2-AES).

![Processeur](VOTRE_SCREENSHOT_132010_ICI)

**Onglet Mémoire** — 2048 MiB de RAM.

![Mémoire](VOTRE_SCREENSHOT_132046_ICI)

**Onglet Réseau** — pont `vmbr0`, modèle Intel E1000, pare-feu activé.

![Réseau](VOTRE_SCREENSHOT_132108_ICI)

**Onglet Confirmation** — récapitulatif complet avant création.

![Confirmation](VOTRE_SCREENSHOT_132127_ICI)

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

Au premier lancement, Portainer affiche un assistant de configuration (*Environment Wizard*) qui propose de se connecter à l'environnement Docker local ou d'en ajouter d'autres.

![Portainer — Quick Setup](VOTRE_SCREENSHOT_PORTAINER_SETUP_ICI)

Une fois configuré, l'onglet **Stacks** liste tous les docker-compose déployés sur la machine : `docker` (Portainer + Nextcloud + Nginx), `homepage`, `ollama-webui`, `pihole` et `uptime`.

![Portainer — Liste des Stacks](VOTRE_SCREENSHOT_PORTAINER_STACKS_ICI)

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
      - "81:81" # Interface Nginx
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

**Pi-Hole** est un bloqueur de publicités et de trackers au niveau réseau. Il agit comme un serveur DNS local et intercepte les requêtes vers les domaines publicitaires pour tous les appareils connectés au réseau domestique — sans aucune configuration nécessaire appareil par appareil.

**Accès :** `http://ip/admin`

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"
    environment:
      TZ: 'Europe/Paris'
      WEBPASSWORD: 'ton_mot_de_passe'
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    restart: unless-stopped
```

```bash
sudo docker compose up -d

# En cas de problème avec le mot de passe
docker exec -it pihole pihole setpassword
```

![Pi-Hole](VOTRE_SCREENSHOT_PIHOLE_ICI)

---

## Acer Gateway DT55 — NAS Debian

L'Acer Gateway DT55 tourne sous **Debian 13 sans interface graphique** pour économiser les ressources (4 Go de RAM). Il joue le rôle de serveur de fichiers pour l'ensemble du réseau local grâce à ses **2,5 To de stockage** répartis sur trois disques durs.

![Acer Gateway DT55](VOTRE_SCREENSHOT_ACER_ICI)

---

### Choix du stockage — LVM plutôt que RAID ou NAS

L'Acer dispose de **trois disques HDD** : 2 × 1 To + 1 × 500 Go, soit 2,5 To au total.

Deux alternatives ont été écartées au profit de **LVM (Logical Volume Manager)** :

**RAID 0 écarté** — le RAID 0 strippe les données en répartissant chaque bloc sur l'ensemble des disques. Avec des disques de tailles différentes, la capacité utilisable est limitée à la taille du plus petit disque multipliée par le nombre de disques : `3 × 500 Go = 1,5 To`. Cela aurait entraîné une **perte de 1 To** de capacité.

**Solution NAS dédiée écartée** — une appliance NAS aurait servi uniquement au stockage, sans réelle possibilité d'apprentissage technique approfondi.

**LVM choisi** — le LVM regroupe les trois disques dans un seul groupe de volumes (*Volume Group*) non chiffré, permettant d'exploiter l'intégralité des **2,5 To** sans perte. C'est aussi une technologie incontournable en administration système Linux, bien plus intéressante à maîtriser dans le cadre du **BTS SIO**. Samba vient ensuite exposer ce volume sur le réseau local de façon **multiplateforme** (Windows, macOS, Linux).

---

### Création du LVM

```bash
# Identifier les disques disponibles
lsblk

# Initialiser chaque disque comme volume physique LVM
sudo pvcreate /dev/sdb /dev/sdc /dev/sdd

# Créer un groupe de volumes regroupant les 3 disques
sudo vgcreate vg-nas /dev/sdb /dev/sdc /dev/sdd

# Créer un volume logique utilisant tout l'espace disponible
sudo lvcreate -l 100%FREE -n lv-stockage vg-nas

# Formater en ext4
sudo mkfs.ext4 /dev/vg-nas/lv-stockage

# Créer le point de montage
sudo mkdir -p /mnt/stockage

# Monter le volume
sudo mount /dev/vg-nas/lv-stockage /mnt/stockage

# Rendre le montage permanent au démarrage
echo '/dev/vg-nas/lv-stockage /mnt/stockage ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

![LVM créé](VOTRE_SCREENSHOT_LVM_ICI)

---

### Samba

**Samba** est un service de partage de fichiers multiplateforme basé sur le protocole SMB/CIFS. Il rend le volume LVM de 2,5 To accessible comme un lecteur réseau depuis n'importe quel appareil du réseau local — Windows, macOS ou Linux — sans logiciel supplémentaire côté client.

**Accès :** `\\ip\stockage` (Windows) ou `smb://ip/stockage` (macOS/Linux)

```bash
# Installation de Samba
sudo apt install samba -y

# Sauvegarder la configuration par défaut
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak

# Éditer la configuration
sudo nano /etc/samba/smb.conf
```

```ini
# Ajouter à la fin du fichier smb.conf
[stockage]
   path = /mnt/stockage
   browseable = yes
   read only = no
   guest ok = no
   valid users = VOTRE_UTILISATEUR
```

```bash
# Créer l'utilisateur Samba
sudo smbpasswd -a VOTRE_UTILISATEUR

# Redémarrer Samba
sudo systemctl restart smbd

# Vérifier que Samba tourne
sudo systemctl status smbd
```

![Samba](VOTRE_SCREENSHOT_SAMBA_ICI)

---

### Webmin

**Webmin** est une interface web d'administration système pour Linux. Il donne une vue graphique complète sur l'Acer Gateway DT55 depuis n'importe quel navigateur : état des composants (CPU, RAM, température), utilisation du stockage, gestion des disques et du LVM, services actifs, logs système et utilisateurs. C'est l'équivalent d'un panneau de contrôle distant sans avoir besoin d'ouvrir un terminal SSH pour chaque vérification.

**Accès :** `https://ip:10000`

```bash
# Télécharger et lancer le script d'installation officiel
curl -o setup-repos.sh https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh
sudo sh setup-repos.sh

# Installer Webmin
sudo apt install webmin -y

# Vérifier que le service tourne
sudo systemctl status webmin
```

![Webmin](VOTRE_SCREENSHOT_WEBMIN_ICI)

---

## Conclusion

Ce homelab repose sur une séparation claire des rôles : le HP EliteDesk concentre toute la puissance de calcul et l'orchestration Docker via Proxmox, tandis que l'Acer Gateway joue le rôle de serveur de fichiers avec ses 2,5 To agrégés en LVM. Cette architecture permet d'héberger des services personnels complets — cloud, IA locale, monitoring, reverse proxy, blocage publicitaire réseau — tout en gardant la main sur ses données et en mettant en pratique des compétences directement liées au parcours **BTS SIO**.

---

*Réalisé par [Yacine Harrache](https://github.com/yacinehrc) — BTS SIO SLAM | EPSI Lille*

---

*Réalisé par [Yacine Harrache](https://github.com/yacinehrc) — BTS SIO SLAM | EPSI Lille*
