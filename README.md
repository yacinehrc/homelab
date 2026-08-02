# Homelab

<img width="1219" height="686" alt="Capture d&#39;écran 2026-08-02 175725" src="https://github.com/user-attachments/assets/6e7e3bed-fe5a-4c8b-93cc-d83e48c637a4" />

> Infrastructure personnelle auto-hébergée sur deux machines physiques — un HP EliteDesk 800 G3 Mini sous Proxmox VE pilotant une VM Docker avec tous les services, et un Acer Gateway DT55 sous Debian servant de stockage avec 2,5 To de stockage via LVM et le partage de fichiers Samba.

---

## Sommaire

- [Architecture](#architecture)
- [HP EliteDesk 800 G3 Mini — Proxmox + Docker](#hp-elitedesk-800-g3-mini--proxmox--docker)
  - [Proxmox VE — Création de la VM Docker](#proxmox-ve--création-de-la-vm-docker)
  - [Installation de Docker](#installation-de-docker)
  - [Portainer](#portainer)
  - [Nginx Proxy Manager](#nginx-proxy-manager)
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

| Machine | OS | Rôle |
|---|---|---|
| **HP EliteDesk 800 G3 Mini** | Proxmox VE (Hyperviseur T1) | VM Docker — tous les services |
| **Acer Gateway DT55** | Debian 13 (sans GUI) | NAS — partage de fichiers 2,5 To |
| **HP 17-cn3078nf** | Windows 11 & Ubuntu | PC principal — gestion SSH & navigateur |

---

## HP EliteDesk 800 G3 Mini — Proxmox + Docker

Le HP EliteDesk tourne sous **Proxmox VE**, un hyperviseur de type 1 qui permet de créer et gérer des machines virtuelles directement sur le matériel. Une VM **Debian 13** héberge l'ensemble des services Docker.


<img width="1919" height="949" alt="Capture d&#39;écran 2026-07-25 160735" src="https://github.com/user-attachments/assets/2f0b63c3-de10-4232-9358-0963b1874195" />


### Proxmox VE — Création de la VM Docker

La VM est créée depuis l'interface web Proxmox avec les paramètres suivants.

**Onglet Général** — nom `VM-Docker`, VM ID `100`, nœud `proxmox`.


<img width="777" height="581" alt="Capture d&#39;écran 2026-07-26 131832" src="https://github.com/user-attachments/assets/8118e54c-e844-45a2-8cc3-12bea7f287e3" />


**Onglet Système d'exploitation** — image ISO `debian-13.6.0-amd64-netinst`, type Linux.


<img width="777" height="591" alt="Capture d&#39;écran 2026-07-26 131728" src="https://github.com/user-attachments/assets/0fedec49-ac54-4feb-b3ef-162606de7761" />


**Onglet Système** — BIOS SeaBIOS, machine i440fx, contrôleur SCSI VirtIO.


<img width="782" height="586" alt="Capture d&#39;écran 2026-07-26 131821" src="https://github.com/user-attachments/assets/fb76094e-b46f-4c16-8bc8-78a53aba28c1" />


**Onglet Disques** — disque IDE de 30 GiB sur `local-lvm`.


<img width="777" height="588" alt="Capture d&#39;écran 2026-07-26 131918" src="https://github.com/user-attachments/assets/7ac9c124-569d-4e3b-8972-ac831d53846b" />


**Onglet Processeur** — 1 socket, 2 cœurs (x86-64-v2-AES).


<img width="774" height="588" alt="Capture d&#39;écran 2026-07-26 132010" src="https://github.com/user-attachments/assets/165b008e-1a62-4576-8e0b-c4c7dc9cca89" />


**Onglet Mémoire** — 2048 MiB de RAM.


<img width="769" height="588" alt="Capture d&#39;écran 2026-07-26 132046" src="https://github.com/user-attachments/assets/e146eaeb-d621-44bf-8302-7d358eea2266" />


**Onglet Réseau** — pont `vmbr0`, modèle Intel E1000, pare-feu activé.


<img width="774" height="582" alt="Capture d&#39;écran 2026-07-26 132108" src="https://github.com/user-attachments/assets/1109ee2f-9dd3-4de5-a8d1-188cc1037680" />


**Onglet Confirmation** — récapitulatif complet avant création.


<img width="774" height="598" alt="Capture d&#39;écran 2026-07-26 132127" src="https://github.com/user-attachments/assets/49db7ef5-93f7-4d3f-bb84-1aa86833fcee" />


---

### Installation de Docker

Installation de la version officielle Docker CE sur la VM Debian 13, via le dépôt officiel Docker.

```bash
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

<img width="1919" height="946" alt="Capture d&#39;écran 2026-08-02 161420" src="https://github.com/user-attachments/assets/dfdb44ec-2c0f-43ba-b746-29cb6c6d7942" />

Une fois configuré, l'onglet **Stacks** liste tous les docker-compose déployés sur la machine : `docker` (Portainer + Nextcloud + Nginx), `homepage`, `ollama-webui`, `pihole` et `uptime`.

<img width="1919" height="950" alt="Capture d&#39;écran 2026-08-02 161459" src="https://github.com/user-attachments/assets/02be90a2-7dfd-4ebd-a552-cd97155956c6" />

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

<img width="1919" height="954" alt="Capture d&#39;écran 2026-08-02 160148" src="https://github.com/user-attachments/assets/ce81b68c-8de9-469e-999d-f7723c87fb7b" />

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

<img width="1917" height="948" alt="Capture d&#39;écran 2026-08-02 155955" src="https://github.com/user-attachments/assets/d36d1f72-d6f5-445f-a487-c9f7c851bcd3" />

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

<img width="1919" height="950" alt="Capture d&#39;écran 2026-08-02 155926" src="https://github.com/user-attachments/assets/f2cb1a47-004c-4ca4-9f12-0b022490a2dc" />

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

<img width="1919" height="954" alt="Capture d&#39;écran 2026-08-02 155935" src="https://github.com/user-attachments/assets/646cb667-9dd6-4566-99c3-1a4f921e4171" />

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

<img width="1919" height="947" alt="Capture d&#39;écran 2026-08-02 123124" src="https://github.com/user-attachments/assets/03a51792-2bdd-4cd5-81f6-689acedd4892" />

---

## Acer Gateway DT55 — NAS Debian

L'Acer Gateway DT55 tourne sous **Debian 13 sans interface graphique** pour économiser les ressources (4 Go de RAM). Il joue le rôle de serveur de fichiers pour l'ensemble du réseau local grâce à ses **2,5 To de stockage** répartis sur trois disques durs (500Go d'origine + 2 × 1 To rajoutés).

---

### Choix du stockage — LVM plutôt que RAID ou NAS

L'Acer dispose de **trois disques HDD** : 2 × 1 To + 1 × 500 Go, soit 2,5 To au total.

Deux alternatives ont été écartées au profit de **LVM (Logical Volume Manager)** :

→ **RAID 0 écarté** — le RAID 0 strippe les données en répartissant chaque bloc sur l'ensemble des disques. Avec des disques de tailles différentes, la capacité utilisable est limitée à la taille du plus petit disque multipliée par le nombre de disques : `3 × 500 Go = 1,5 To`. Cela aurait entraîné une **perte de 1 To** de capacité.

→ **Solution NAS dédiée écartée** — une appliance NAS aurait servi uniquement au stockage, sans réelle possibilité d'apprentissage technique approfondi pour mes études.

→ **LVM choisi** — le LVM regroupe les trois disques dans un seul groupe de volumes (*Volume Group*) non chiffré, permettant d'exploiter l'intégralité des **2,5 To** sans perte. C'est aussi une technologie incontournable en administration système Linux, bien plus intéressante à maîtriser dans le cadre du **BTS SIO**. Samba vient ensuite exposer ce volume sur le réseau local de façon **multiplateforme** (Windows, macOS, Linux).

---

### Création du LVM

Pour la création du LVM, je l'ai fait pendant l'installation de Debian 13, j'ai sélectionné un partitionnement manuel, regroupé mes disques dans un groupe de volumes (VG), puis créé un volume logique (LV) à l'intérieur pour y assigner le point de montage de la racine (/).

---

### Samba

**Samba** est un service de partage de fichiers multiplateforme basé sur le protocole SMB/CIFS. Il rend le volume LVM de 2,5 To accessible comme un lecteur réseau depuis n'importe quel appareil du réseau local — Windows, macOS ou Linux — sans logiciel supplémentaire côté client.

**Accès :** `\\ip\NOM` *Nom que vous donnerez lors de la configuration de Samba dans* ```bash sudo nano /etc/samba/smb.conf``` (Windows) ou `smb://ip/NOM` (macOS/Linux)

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

<img width="775" height="234" alt="Capture d&#39;écran 2026-08-02 170848" src="https://github.com/user-attachments/assets/48e79d81-71b2-448b-a801-874018f57bc5" />

Suite à une réinstallation complète de Debian sur la même adresse IP, des conflits de cache d'authentification ont initialement compromis la connexion au service Samba. La résolution de cet incident a nécessité une intervention directe sur le gestionnaire d'identités pour réattribuer explicitement les droits d'accès aux partages.

<img width="1077" height="516" alt="Capture d&#39;écran 2026-08-02 171447" src="https://github.com/user-attachments/assets/c4b4ff49-9516-48e4-b679-d45a5775fc3e" />

---

### Webmin

**Webmin** est une interface web d'administration système pour Linux, tout comme Cockpit mais l'objectif est de varier les outils. Il donne une vue graphique complète sur l'Acer Gateway DT55 depuis n'importe quel navigateur : état des composants (CPU, RAM, température), utilisation du stockage, gestion des disques et du LVM, services actifs, logs système et utilisateurs. C'est l'équivalent d'un panneau de contrôle distant sans avoir besoin d'ouvrir un terminal SSH pour chaque vérification.

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

<img width="1919" height="953" alt="Capture d&#39;écran 2026-08-02 165523" src="https://github.com/user-attachments/assets/ba95121d-d77e-47b5-af39-490dbbcebaff" />

---

## Conclusion

Ce homelab repose sur une séparation claire des rôles : le HP EliteDesk concentre toute la puissance de calcul et l'orchestration Docker via Proxmox, tandis que l'Acer Gateway joue le rôle de serveur de fichiers avec ses 2,5 To agrégés en LVM. Cette architecture permet d'héberger des services personnels complets — cloud, IA locale, monitoring, reverse proxy, blocage publicitaire réseau — tout en gardant la main sur ses données et en mettant en pratique des compétences directement liées au parcours **BTS SIO**.

---

*Réalisé par [Yacine Harrache](https://github.com/yacinehrc) — BTS SIO SLAM | EPSI Lille*
