# Guide Avancé : Streaming Live et Haute Disponibilité avec Nginx

Ce guide détaille deux configurations avancées de Nginx :

- **A. Streaming live avec nginx-rtmp-module**
- **B. Haute disponibilité avec Keepalived**

---

## Table des matières

1. [A. Streaming Live avec RTMP](#a-streaming-live-avec-rtmp)

   - [Installation des dépendances](#1-installation-des-dépendances)
   - [Compilation de Nginx](#2-compilation-de-nginx)
   - [Configuration RTMP](#3-configuration-rtmp)
   - [Démarrage du serveur](#4-démarrage-du-serveur)

2. [B. Haute Disponibilité avec Keepalived](#b-haute-disponibilité-avec-keepalived)

   - [Principe de fonctionnement](#principe-de-fonctionnement)
   - [Installation](#1-installation-de-keepalived)
   - [Configuration Master/Backup](#2-configuration-masterbackup)
   - [Tests et vérification](#3-tests-et-vérification)

3. [C. Health Checks Automatisés](#c-health-checks-automatisés)

---

## A. Streaming Live avec RTMP

### Contexte

Le **nginx-rtmp-module** permet à Nginx de gérer des flux vidéo en temps réel (streaming live) via le protocole **RTMP** (Real-Time Messaging Protocol) et de les redistribuer en **HLS** (HTTP Live Streaming) pour une compatibilité maximale avec les navigateurs et appareils mobiles.

> **Note** : Cette méthode nécessite de compiler Nginx depuis les sources, ce qui peut créer des conflits avec une installation via `apt`. Elle est idéale pour comprendre le processus de compilation.

---

### 1. Installation des dépendances

```bash
apt update
apt install -y build-essential libpcre3 libpcre3-dev libssl-dev zlib1g-dev git wget
```

#### Explication des bibliothèques :

| Bibliothèque                    | Rôle                                                                                                                          |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **build-essential**             | Contient GCC, G++, make et autres outils nécessaires pour compiler du code C/C++                                              |
| **libpcre3** / **libpcre3-dev** | **PCRE** (Perl Compatible Regular Expressions) : permet à Nginx d'utiliser des expressions régulières dans les configurations |
| **libssl-dev**                  | Support SSL/TLS pour HTTPS                                                                                                    |
| **zlib1g-dev**                  | Bibliothèque de compression gzip pour compresser les réponses HTTP                                                            |
| **git**                         | Pour cloner le dépôt nginx-rtmp-module                                                                                        |
| **wget**                        | Pour télécharger l'archive Nginx                                                                                              |

---

### 2. Téléchargement des sources

```bash
cd /tmp
git clone https://github.com/arut/nginx-rtmp-module.git
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar xzf nginx-1.24.0.tar.gz
cd nginx-1.24.0
```

#### Explication :

- **nginx-rtmp-module** : Module tiers développé par Roman Arutyunyan pour ajouter le support RTMP à Nginx
- **nginx-1.24.0** : Version stable de Nginx (vous pouvez utiliser une version plus récente si disponible)
- **tar xzf** : Décompresse l'archive (.tar.gz)

---

### 3. Compilation de Nginx

```bash
./configure --with-http_ssl_module --add-module=../nginx-rtmp-module
make -j$(nproc)
make install
```

#### Explication des commandes :

| Commande                            | Description                                                                   |
| ----------------------------------- | ----------------------------------------------------------------------------- |
| `./configure`                       | Script de configuration qui vérifie les dépendances et prépare la compilation |
| `--with-http_ssl_module`            | Active le support HTTPS/SSL dans Nginx                                        |
| `--add-module=../nginx-rtmp-module` | Ajoute le module RTMP en tant que module compilé statiquement                 |
| `make -j$(nproc)`                   | Compile Nginx en parallèle (utilise tous les cœurs CPU disponibles)           |
| `make install`                      | Installe Nginx dans `/usr/local/nginx` par défaut                             |

#### Structure d'installation :

```
/usr/local/nginx/
├── conf/          # Fichiers de configuration
│   └── nginx.conf
├── html/          # Fichiers web statiques
├── logs/          # Logs d'accès et d'erreur
└── sbin/          # Binaire nginx
    └── nginx
```

---

### 4. Configuration RTMP

Créez ou modifiez `/usr/local/nginx/conf/nginx.conf` :

```nginx
worker_processes auto;
events {
    worker_connections 1024;
}

# ============================
# BLOC RTMP (Streaming Live)
# ============================
rtmp {
    server {
        listen 1935;              # Port RTMP standard
        chunk_size 4096;          # Taille des chunks RTMP

        # Application de streaming live
        application live {
            live on;              # Active le streaming en direct
            record off;           # Désactive l'enregistrement

            # Conversion RTMP → HLS
            hls on;                           # Active HLS
            hls_path /var/www/html/hls;       # Dossier pour les segments HLS
            hls_fragment 3s;                  # Durée d'un segment vidéo
            hls_playlist_length 60s;          # Durée totale de la playlist
        }
    }
}

# ============================
# BLOC HTTP (Distribution HLS)
# ============================
http {
    include mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        server_name localhost;

        # Servir les segments HLS
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /var/www/html;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;  # CORS pour lecteurs web
        }

        # Page d'accueil
        location / {
            root html;
            index index.html;
        }
    }
}
```

#### Explication des directives RTMP :

| Directive                 | Rôle                                                          |
| ------------------------- | ------------------------------------------------------------- |
| `listen 1935`             | Port d'écoute RTMP (standard pour le streaming)               |
| `chunk_size 4096`         | Taille des chunks de données RTMP (optimisation réseau)       |
| `live on`                 | Active le mode streaming en direct                            |
| `record off`              | Désactive l'enregistrement des flux sur disque                |
| `hls on`                  | Active la conversion RTMP → HLS                               |
| `hls_path`                | Répertoire où stocker les segments `.ts` et playlists `.m3u8` |
| `hls_fragment 3s`         | Chaque segment vidéo dure 3 secondes                          |
| `hls_playlist_length 60s` | La playlist contient 60 secondes de vidéo (20 segments × 3s)  |

#### Créer le dossier HLS :

```bash
mkdir -p /var/www/html/hls
chown -R www-data:www-data /var/www/html/hls  # Si nginx tourne sous www-data
```

---

### 5. Démarrage du serveur

```bash
# Créer le répertoire pour les logs si nécessaire
mkdir -p /usr/local/nginx/logs

# Démarrer Nginx
/usr/local/nginx/sbin/nginx

# Vérifier le processus
ps aux | grep nginx

# Voir les logs
tail -f /usr/local/nginx/logs/error.log
tail -f /usr/local/nginx/logs/access.log
```

#### Commandes utiles :

```bash
# Recharger la configuration sans arrêter le serveur
/usr/local/nginx/sbin/nginx -s reload

# Arrêter proprement
/usr/local/nginx/sbin/nginx -s quit

# Arrêter immédiatement
/usr/local/nginx/sbin/nginx -s stop

# Tester la configuration
/usr/local/nginx/sbin/nginx -t
```

---

### 6. Tester le streaming

#### Publier un flux avec OBS Studio :

1. **Serveur** : `rtmp://<VOTRE_IP>/live`
2. **Clé de flux** : `test` (ou n'importe quel nom)

#### URL de lecture HLS :

```
http://<VOTRE_IP>/hls/test.m3u8
```

Ouvrez cette URL dans un lecteur compatible HLS :

- **VLC** : Fichier → Ouvrir un flux réseau
- **Navigateur** : utilisez un lecteur JavaScript comme **Video.js** ou **HLS.js**

---

## B. Haute Disponibilité avec Keepalived

### Principe de fonctionnement

**Keepalived** implémente le protocole **VRRP** (Virtual Router Redundancy Protocol) pour créer une **IP virtuelle (VIP)** partagée entre plusieurs serveurs. En cas de panne du serveur maître, la VIP bascule automatiquement vers le serveur de secours.

#### Architecture :

```
                      VIP: 192.168.1.100
                              |
              +---------------+---------------+
              |                               |
         [MASTER]                        [BACKUP]
      Node1: 192.168.1.10           Node2: 192.168.1.11
      Priority: 150                 Priority: 100

      Clients → 192.168.1.100 (VIP)
```

#### Fonctionnement :

1. Le **MASTER** possède la VIP et répond aux requêtes
2. Le MASTER envoie des annonces VRRP périodiques (heartbeat)
3. Si le BACKUP ne reçoit plus d'annonces, il prend la VIP
4. Quand le MASTER revient, il reprend la VIP (selon priorité)

---

### Prérequis

- **2 machines/VM** sur le même réseau local
- **IP fixe** pour chaque machine
- **Nginx installé** sur les deux machines

---

### 1. Installation de Keepalived

Sur **node1 (master)** et **node2 (backup)** :

```bash
apt update
apt install -y keepalived
```

---

### 2. Identifier l'interface réseau

Sur chaque machine :

```bash
ip addr show
```

Exemple de sortie :

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP
    inet 192.168.1.10/24 brd 192.168.1.255 scope global ens33
```

**Note l'interface** : `ens33` (ou `eth0`, `enp0s3`, etc.)

---

### 3. Configuration Master (node1)

Éditez `/etc/keepalived/keepalived.conf` sur **node1** (IP: 192.168.1.10) :

```bash
vrrp_instance VI_1 {
    state MASTER                  # Rôle initial : MASTER
    interface ens33               # Interface réseau (à adapter)
    virtual_router_id 51          # ID du routeur virtuel (doit être identique sur tous les nœuds)
    priority 150                  # Priorité élevée = préférence pour être MASTER
    advert_int 1                  # Intervalle entre les annonces VRRP (en secondes)

    authentication {
        auth_type PASS            # Type d'authentification
        auth_pass mysecretpass    # Mot de passe partagé (8 caractères max)
    }

    virtual_ipaddress {
        192.168.1.100/24          # VIP (choisir une IP libre du réseau)
    }
}
```

#### Explication des paramètres :

| Paramètre           | Description                                                                     |
| ------------------- | ------------------------------------------------------------------------------- |
| `state MASTER`      | Rôle initial (sera élu automatiquement selon priorité)                          |
| `interface`         | Interface réseau sur laquelle attacher la VIP                                   |
| `virtual_router_id` | Identifiant VRRP (0-255), doit être **identique** sur tous les nœuds du cluster |
| `priority`          | Plus la valeur est élevée, plus le nœud a de chances d'être MASTER (0-255)      |
| `advert_int`        | Fréquence des annonces VRRP (1 seconde = réactivité rapide)                     |
| `auth_type PASS`    | Authentification par mot de passe simple                                        |
| `auth_pass`         | Mot de passe partagé entre tous les nœuds                                       |
| `virtual_ipaddress` | La VIP flottante que les clients utiliseront                                    |

---

### 4. Configuration Backup (node2)

Éditez `/etc/keepalived/keepalived.conf` sur **node2** (IP: 192.168.1.11) :

```bash
vrrp_instance VI_1 {
    state BACKUP                  # Rôle initial : BACKUP
    interface ens33               # Même interface que le master
    virtual_router_id 51          # DOIT être identique au master
    priority 100                  # Priorité plus basse que le master
    advert_int 1                  # Même intervalle que le master

    authentication {
        auth_type PASS
        auth_pass mysecretpass    # DOIT être identique au master
    }

    virtual_ipaddress {
        192.168.1.100/24          # Même VIP que le master
    }
}
```

**Différences par rapport au master :**

- `state BACKUP` : rôle de secours
- `priority 100` : priorité inférieure (le master a 150)

---

### 5. Démarrer Keepalived

Sur **les deux nœuds** :

```bash
# Activer au démarrage et lancer le service
systemctl enable --now keepalived

# Vérifier le statut
systemctl status keepalived

# Voir les logs en temps réel
journalctl -u keepalived -f
```

---

### 6. Vérification et tests

#### a) Vérifier la VIP sur le MASTER

Sur **node1** :

```bash
ip addr show ens33
```

Vous devriez voir la VIP ajoutée :

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.1.10/24 brd 192.168.1.255 scope global ens33
    inet 192.168.1.100/24 scope global secondary ens33  ← VIP présente
```

#### b) Vérifier que la VIP n'est PAS sur le BACKUP

Sur **node2** :

```bash
ip addr show ens33
```

Vous ne devriez voir que l'IP réelle (192.168.1.11), pas la VIP.

#### c) Tester le basculement (failover)

1. **Arrêter keepalived sur le master** :

```bash
# Sur node1
systemctl stop keepalived
```

2. **Vérifier la VIP sur le backup** :

```bash
# Sur node2
ip addr show ens33
```

→ La VIP doit maintenant apparaître sur **node2** (généralement en 2-3 secondes).

3. **Tester la connectivité** :

```bash
ping 192.168.1.100
curl http://192.168.1.100
```

4. **Redémarrer le master** :

```bash
# Sur node1
systemctl start keepalived
```

→ La VIP devrait revenir sur **node1** (selon la priorité).

---

### 7. Utilisation avec Nginx

#### Configuration identique sur les deux nœuds :

1. **Synchroniser les configurations** :

```bash
# Depuis node1 vers node2
rsync -av /etc/nginx/ root@192.168.1.11:/etc/nginx/

# Ou utiliser un dépôt Git pour versionner les configs
```

2. **Synchroniser les contenus** :

```bash
# Partager /var/www/html via NFS, GlusterFS, ou rsync périodique
```

3. **Les clients se connectent à la VIP** :

```bash
http://192.168.1.100
```

#### Avantages :

- **Transparence** : les clients ne voient qu'une seule IP
- **Disponibilité** : basculement automatique en cas de panne
- **Aucune configuration DNS** nécessaire

---

## C. Health Checks Automatisés

### Objectif

Détecter automatiquement si Nginx est en panne sur le MASTER et déclencher un basculement même si le serveur reste allumé.

---

### 1. Script de vérification

Créez `/usr/local/bin/check_nginx.sh` :

```bash
#!/bin/bash
# Script de health check pour Nginx
# Retourne 0 si Nginx répond correctement, sinon code d'erreur

# Tester si Nginx répond sur le port 80
curl -sS http://127.0.0.1/ > /dev/null

# $? contient le code de retour de curl
# 0 = succès, autre = échec
if [ $? -eq 0 ]; then
    exit 0  # Nginx fonctionne
else
    exit 1  # Nginx ne répond pas
fi
```

#### Rendre le script exécutable :

```bash
chmod +x /usr/local/bin/check_nginx.sh

# Tester manuellement
/usr/local/bin/check_nginx.sh
echo $?  # Doit afficher 0 si Nginx fonctionne
```

---

### 2. Intégration dans Keepalived

Modifiez `/etc/keepalived/keepalived.conf` :

```bash
# Définir le script de vérification
vrrp_script chk_nginx {
    script "/usr/local/bin/check_nginx.sh"  # Chemin du script
    interval 2                                # Vérifier toutes les 2 secondes
    weight -20                                # Pénalité de priorité en cas d'échec
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass mysecretpass
    }

    # Activer le suivi du script
    track_script {
        chk_nginx  # Référence au script défini plus haut
    }

    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

#### Explication :

| Paramètre      | Description                                           |
| -------------- | ----------------------------------------------------- |
| `script`       | Chemin vers le script de vérification                 |
| `interval 2`   | Exécuter le script toutes les 2 secondes              |
| `weight -20`   | Si le script échoue, réduire la priorité de 20 points |
| `track_script` | Active le suivi du script dans l'instance VRRP        |

#### Comportement :

- Si le script échoue, la priorité du MASTER passe de **150 à 130** (150 - 20)
- Si le BACKUP a une priorité de **100**, le MASTER reste maître
- Mais si le script échoue plusieurs fois ou si vous configurez `weight -51`, la priorité tombe à **99** → le BACKUP prend la VIP

---

### 3. Script avancé avec vérification multiple

```bash
#!/bin/bash
# Health check avancé pour Nginx

# Vérifier si le processus Nginx tourne
if ! pgrep -x nginx > /dev/null; then
    echo "Nginx process not found"
    exit 1
fi

# Vérifier si Nginx répond sur HTTP
if ! curl -sf http://127.0.0.1/ > /dev/null; then
    echo "Nginx not responding on port 80"
    exit 1
fi

# Vérifier si Nginx répond sur HTTPS (si configuré)
if ! curl -sf -k https://127.0.0.1/ > /dev/null 2>&1; then
    echo "Nginx not responding on port 443"
    exit 1
fi

# Tout est OK
exit 0
```

---

### 4. Notifications par email (optionnel)

Vous pouvez configurer Keepalived pour envoyer des emails lors des transitions :

```bash
global_defs {
    notification_email {
        admin@example.com
        ops@example.com
    }
    notification_email_from keepalived@node1
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
}

vrrp_instance VI_1 {
    # ... configuration existante ...

    # Scripts de notification
    notify_master "/usr/local/bin/notify_master.sh"
    notify_backup "/usr/local/bin/notify_backup.sh"
    notify_fault "/usr/local/bin/notify_fault.sh"
}
```

Exemple de script `/usr/local/bin/notify_master.sh` :

```bash
#!/bin/bash
echo "$(date): Node $(hostname) is now MASTER" | mail -s "Keepalived: MASTER" admin@example.com
```

---

## Résumé des concepts clés

### RTMP & HLS

| Concept      | Description                                                                  |
| ------------ | ---------------------------------------------------------------------------- |
| **RTMP**     | Protocole de streaming temps réel, utilisé pour l'ingestion (OBS → Serveur)  |
| **HLS**      | Protocole de streaming HTTP, compatible tous navigateurs (Serveur → Clients) |
| **Segment**  | Fragment vidéo de quelques secondes (.ts)                                    |
| **Playlist** | Fichier .m3u8 listant les segments disponibles                               |

### Keepalived & VRRP

| Concept          | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| **VIP**          | IP virtuelle flottante partagée entre plusieurs serveurs     |
| **VRRP**         | Protocole d'élection automatique du serveur actif            |
| **Priority**     | Poids pour déterminer quel serveur devient MASTER            |
| **Track script** | Script de surveillance pour détecter les pannes applicatives |
| **Failover**     | Basculement automatique vers le serveur de secours           |

---

## Dépannage

### Problème : La VIP n'apparaît pas

```bash
# Vérifier les logs Keepalived
journalctl -u keepalived -n 50

# Vérifier les règles firewall
iptables -L -n -v
# VRRP utilise le protocole 112, assurez-vous qu'il est autorisé

# Autoriser VRRP
iptables -A INPUT -p vrrp -j ACCEPT
```

### Problème : Nginx ne démarre pas après compilation

```bash
# Vérifier les erreurs de configuration
/usr/local/nginx/sbin/nginx -t

# Vérifier les dépendances manquantes
ldd /usr/local/nginx/sbin/nginx

# Vérifier les permissions
ls -la /usr/local/nginx/logs/
```

### Problème : Les segments HLS ne sont pas créés

```bash
# Vérifier les permissions du dossier HLS
ls -la /var/www/html/hls/
chown -R www-data:www-data /var/www/html/hls/

# Vérifier les logs Nginx
tail -f /usr/local/nginx/logs/error.log

# Tester la publication RTMP
ffmpeg -re -i video.mp4 -c copy -f flv rtmp://localhost/live/test
```

---

## Ressources supplémentaires

- **Documentation Nginx RTMP Module** : https://github.com/arut/nginx-rtmp-module
- **Documentation Keepalived** : https://www.keepalived.org/documentation.html
- **RFC 5798 (VRRP)** : https://tools.ietf.org/html/rfc5798
- **HLS Specification** : https://datatracker.ietf.org/doc/html/rfc8216

---

## Auteur

Guide créé pour le TP Network Configuration avec Nginx.

**Date** : Novembre 2024
