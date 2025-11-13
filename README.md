# Configuration Nginx - Reverse Proxy, Load Balancer et Forward Proxy

## 📋 Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Architecture globale](#architecture-globale)
3. [Étape 1 : Préparation des backends](#étape-1--préparation-des-backends)
4. [Étape 2 : Configuration Nginx (Reverse Proxy + Load Balancer)](#étape-2--configuration-nginx-reverse-proxy--load-balancer)
5. [Étape 3 : Configuration Forward Proxy](#étape-3--configuration-forward-proxy)
6. [Étape 4 : Tests et validation](#étape-4--tests-et-validation)
7. [Commandes utiles](#commandes-utiles)
8. [Dépannage](#dépannage)

---

## 🎯 Vue d'ensemble

Ce guide configure un serveur Nginx avec trois fonctionnalités principales :

- **Reverse Proxy** : Masque un backend et expose une URL publique
- **Load Balancer** : Distribue la charge entre plusieurs backends
- **Forward Proxy** : Permet aux clients de sortir vers Internet via le proxy
- **Site statique** : Page d'accueil par défaut

---

## 🏗️ Architecture globale

```
┌─────────────────────────────────────────────────────────────────────┐
│                            SERVEUR VM                                │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      NGINX (Port 80)                         │   │
│  │                                                               │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────────┐ │   │
│  │  │  Site Statique │  │ Reverse Proxy  │  │Load Balancer  │ │   │
│  │  │       /        │  │    /apache/    │  │     /lb/      │ │   │
│  │  │                │  │                │  │               │ │   │
│  │  │ /var/www/      │  │  proxy_pass    │  │  upstream     │ │   │
│  │  │ nginx_site     │  │  → :8080       │  │  pool         │ │   │
│  │  └────────────────┘  └────────┬───────┘  └───────┬───────┘ │   │
│  └──────────────────────────────┼──────────────────┼──────────┘   │
│                                   │                  │              │
│                                   ▼                  ▼              │
│  ┌───────────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │ Backend "Apache"  │   │  Backend 1   │   │  Backend 2   │     │
│  │   Port 8080       │   │  Port 8081   │   │  Port 8082   │     │
│  │                   │   │              │   │              │     │
│  │ Python HTTP       │   │ Python HTTP  │   │ Python HTTP  │     │
│  │ Server            │   │ Server       │   │ Server       │     │
│  └───────────────────┘   └──────────────┘   └──────────────┘     │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              NGINX Forward Proxy (Port 8888)                │   │
│  │                                                              │   │
│  │  Client (VM) → :8888 → Internet (example.com, etc.)        │   │
│  └────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Étape 1 : Préparation des backends

### 🎯 Objectif

Créer 3 serveurs HTTP Python simples qui simulent des applications backend.

### 📝 Commandes

```bash
# Créer les dossiers pour chaque backend
mkdir -p /var/www/backend_apache /var/www/backend1 /var/www/backend2

# Créer les fichiers HTML de test
echo "<h1>Backend APACHE simulé (port 8080)</h1>" > /var/www/backend_apache/index.html
echo "<h1>Backend 1 (port 8081)</h1>" > /var/www/backend1/index.html
echo "<h1>Backend 2 (port 8082)</h1>" > /var/www/backend2/index.html

# Lancer les serveurs HTTP en arrière-plan
nohup python3 -m http.server 8080 --directory /var/www/backend_apache > /var/log/backend8080.log 2>&1 &
nohup python3 -m http.server 8081 --directory /var/www/backend1 > /var/log/backend8081.log 2>&1 &
nohup python3 -m http.server 8082 --directory /var/www/backend2 > /var/log/backend8082.log 2>&1 &
```

### ✅ Vérification

```bash
# Vérifier que les ports sont ouverts
ss -tlnp | grep -E '8080|8081|8082'

# Vérifier les processus
ps aux | grep http.server | grep -v grep

# Tester chaque backend directement
curl http://127.0.0.1:8080/
curl http://127.0.0.1:8081/
curl http://127.0.0.1:8082/
```

### 📊 Schéma des backends

```
┌─────────────────────────────────────────────┐
│         Backends Python HTTP                │
├─────────────────────────────────────────────┤
│                                             │
│  Port 8080 → /var/www/backend_apache/      │
│              "Backend APACHE simulé"        │
│                                             │
│  Port 8081 → /var/www/backend1/            │
│              "Backend 1"                    │
│                                             │
│  Port 8082 → /var/www/backend2/            │
│              "Backend 2"                    │
│                                             │
│  Processus: python3 -m http.server         │
│  Logs: /var/log/backend808*.log            │
└─────────────────────────────────────────────┘
```

---

## 🔄 Étape 2 : Configuration Nginx (Reverse Proxy + Load Balancer)

### 🎯 Objectif

Configurer Nginx pour :

1. Servir un site statique sur `/`
2. Reverse proxy vers le backend 8080 sur `/apache/`
3. Load balancer entre 8081 et 8082 sur `/lb/`

### 📝 Configuration

```bash
# Créer le fichier de configuration
cat > /etc/nginx/sites-available/nginx_site <<'EOF'
# upstream pour load balancer
upstream backend_pool {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 80;
    server_name _;

    # site statique par défaut
    root /var/www/nginx_site;
    index index.html index.htm;

    # racine : site nginx static
    location / {
        try_files $uri $uri/ =404;
    }

    # reverse proxy vers "apache" simulé sur 8080
    location /apache/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    # load balancer (round-robin par défaut)
    location /lb/ {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout 3s;
        proxy_read_timeout 10s;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    }
}
EOF

# Créer le site statique
mkdir -p /var/www/nginx_site
echo "<h1>Bonjour depuis /var/www/nginx_site</h1>" > /var/www/nginx_site/index.html
chown -R www-data:www-data /var/www/nginx_site

# Activer le site
ln -sf /etc/nginx/sites-available/nginx_site /etc/nginx/sites-enabled/nginx_site

# Tester et recharger
nginx -t
systemctl reload nginx
```

### 📊 Schéma Reverse Proxy

```
Client                  Nginx (Port 80)            Backend (Port 8080)
  │                          │                            │
  │  GET /apache/page.html   │                            │
  ├─────────────────────────►│                            │
  │                          │  GET /page.html            │
  │                          ├───────────────────────────►│
  │                          │                            │
  │                          │  HTTP 200 + HTML           │
  │                          │◄───────────────────────────┤
  │  HTTP 200 + HTML         │                            │
  │◄─────────────────────────┤                            │
  │                          │                            │

Avantages:
- Cache l'adresse réelle du backend
- Permet de changer le backend sans affecter l'URL publique
- Ajoute des headers de sécurité et de traçabilité
```

### 📊 Schéma Load Balancer (Round-Robin)

```
                              Nginx (Port 80)
                                    │
                          ┌─────────┴─────────┐
                          │   upstream pool    │
                          │    round-robin     │
                          └─────────┬─────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
          ┌──────────────────┐           ┌──────────────────┐
          │   Backend 1      │           │   Backend 2      │
          │   Port 8081      │           │   Port 8082      │
          └──────────────────┘           └──────────────────┘

Requête 1  →  Backend 1 (8081)
Requête 2  →  Backend 2 (8082)
Requête 3  →  Backend 1 (8081)  ← Round-robin
Requête 4  →  Backend 2 (8082)
Requête 5  →  Backend 1 (8081)

Avantages:
- Distribution équitable de la charge
- Haute disponibilité (si un backend tombe, l'autre prend le relais)
- Scalabilité horizontale (ajouter plus de backends facilement)
```

### 📊 Schéma des 3 routes

```
┌─────────────────────────────────────────────────────────┐
│              Nginx Server (Port 80)                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Route 1: /                                             │
│  ┌────────────────────────────────────────────┐        │
│  │  Site statique                             │        │
│  │  Fichiers: /var/www/nginx_site/            │        │
│  │  Utilisation: Page d'accueil, assets       │        │
│  └────────────────────────────────────────────┘        │
│                                                          │
│  Route 2: /apache/                                      │
│  ┌────────────────────────────────────────────┐        │
│  │  Reverse Proxy                             │        │
│  │  Backend: 127.0.0.1:8080                   │        │
│  │  Utilisation: Masquer backend Apache       │        │
│  └────────────────────────────────────────────┘        │
│                                                          │
│  Route 3: /lb/                                          │
│  ┌────────────────────────────────────────────┐        │
│  │  Load Balancer                             │        │
│  │  Backends:                                 │        │
│  │    - 127.0.0.1:8081 (Backend 1)           │        │
│  │    - 127.0.0.1:8082 (Backend 2)           │        │
│  │  Algorithme: Round-robin                   │        │
│  └────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

---

## 🌐 Étape 3 : Configuration Forward Proxy

### 🎯 Objectif

Permettre aux clients (sur la VM ou réseau local) de sortir vers Internet via Nginx.

### 📝 Configuration

```bash
# Créer la configuration du forward proxy
cat > /etc/nginx/conf.d/forward_proxy.conf <<'EOF'
server {
    listen 8888;
    resolver 8.8.8.8 valid=10s;
    resolver_timeout 5s;

    # autoriser localhost et réseau local
    allow 127.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    location / {
        # forward minimal (HTTP only)
        proxy_pass $scheme://$http_host$request_uri;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 10s;
        proxy_read_timeout 30s;
    }
}
EOF

# Tester et recharger
nginx -t && systemctl reload nginx
```

### 📊 Schéma Forward Proxy

```
Client (VM)              Forward Proxy (Port 8888)      Internet
    │                              │                         │
    │  GET http://example.com      │                         │
    ├─────────────────────────────►│                         │
    │                              │  GET http://example.com │
    │                              ├────────────────────────►│
    │                              │                         │
    │                              │  HTTP 200 + HTML        │
    │                              │◄────────────────────────┤
    │  HTTP 200 + HTML             │                         │
    │◄─────────────────────────────┤                         │
    │                              │                         │

Différence Reverse vs Forward Proxy:

┌────────────────────────────────────────────────────────────┐
│  REVERSE PROXY                                             │
│  Client → Nginx → Backend interne                          │
│  Protège le SERVEUR (cache l'origine)                      │
│  Exemple: /apache/ → port 8080                             │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  FORWARD PROXY                                             │
│  Client interne → Nginx → Internet                         │
│  Protège le CLIENT (filtre, cache, anonymise)              │
│  Exemple: VM → :8888 → example.com                         │
└────────────────────────────────────────────────────────────┘
```

### 🔒 Restrictions de sécurité

```
Règles d'accès du Forward Proxy:
┌─────────────────────────────────────────┐
│  allow 127.0.0.0/8;     ✅ Autorisé    │  ← localhost
│  allow 192.168.0.0/16;  ✅ Autorisé    │  ← réseau local
│  deny all;              ❌ Refusé      │  ← reste du monde
└─────────────────────────────────────────┘

Pourquoi?
- Éviter que le proxy soit utilisé comme proxy public ouvert
- Limiter l'accès au réseau local uniquement
- Prévenir les abus (spam, attaques DDoS via le proxy)
```

---

## ✅ Étape 4 : Tests et validation

### 🧪 Test 1 : Site statique

```bash
curl -s http://127.0.0.1/ | sed -n '1,5p'
```

**Résultat attendu :**

```html
<h1>Bonjour depuis /var/www/nginx_site</h1>
```

### 🧪 Test 2 : Reverse Proxy

```bash
curl -s http://127.0.0.1/apache/ | sed -n '1,10p'
```

**Résultat attendu :**

```html
<h1>Backend APACHE simulé (port 8080)</h1>
```

### 🧪 Test 3 : Load Balancer (Round-Robin)

```bash
for i in {1..6}; do curl -s http://127.0.0.1/lb/ | sed -n '1,3p'; echo "----"; done
```

**Résultat attendu (alternance) :**

```
<h1>Backend 1 (port 8081)</h1>
----
<h1>Backend 2 (port 8082)</h1>
----
<h1>Backend 1 (port 8081)</h1>
----
<h1>Backend 2 (port 8082)</h1>
----
...
```

### 🧪 Test 4 : Forward Proxy

```bash
curl -s -I -x http://127.0.0.1:8888 http://example.com
```

**Résultat attendu :**

```
HTTP/1.1 200 OK
Server: nginx/...
Date: ...
Content-Type: text/html
...
```

### 📊 Tableau récapitulatif des tests

```
┌──────────────────────┬─────────────────────────┬──────────────────────────┐
│  Test                │  URL                    │  Résultat attendu        │
├──────────────────────┼─────────────────────────┼──────────────────────────┤
│  Site statique       │  http://localhost/      │  Page HTML nginx_site    │
│  Reverse proxy       │  http://localhost/apache│  Backend port 8080       │
│  Load balancer       │  http://localhost/lb/   │  Alterne 8081/8082       │
│  Forward proxy       │  -x :8888 example.com   │  Headers HTTP externes   │
└──────────────────────┴─────────────────────────┴──────────────────────────┘
```

---

## 🛠️ Commandes utiles

### 📡 Surveillance des ports

```bash
# Voir tous les ports en écoute
ss -tlnp | egrep '(:80|:8888|:8080|:8081|:8082)'

# Résultat attendu:
# LISTEN 0 511 0.0.0.0:80      *:*     users:(("nginx",...))
# LISTEN 0 511 0.0.0.0:8888    *:*     users:(("nginx",...))
# LISTEN 0 5   0.0.0.0:8080    *:*     users:(("python3",...))
# LISTEN 0 5   0.0.0.0:8081    *:*     users:(("python3",...))
# LISTEN 0 5   0.0.0.0:8082    *:*     users:(("python3",...))
```

### 📋 Logs en temps réel

```bash
# Logs Nginx (accès et erreurs)
tail -f /var/log/nginx/access.log /var/log/nginx/error.log

# Logs backends Python
tail -f /var/log/backend8080.log /var/log/backend8081.log /var/log/backend8082.log
```

### 🔍 Vérification des processus

```bash
# Voir les serveurs Python HTTP
ps aux | grep "http.server" | grep -v grep

# Voir les processus Nginx
ps aux | grep nginx | grep -v grep

# Statut du service Nginx
systemctl status nginx
```

### 🔄 Redémarrage et rechargement

```bash
# Tester la configuration sans redémarrer
nginx -t

# Recharger la configuration (sans interruption)
systemctl reload nginx

# Redémarrer Nginx (interruption brève)
systemctl restart nginx

# Arrêter Nginx
systemctl stop nginx

# Démarrer Nginx
systemctl start nginx
```

### 🛑 Arrêt des backends Python

```bash
# Méthode 1: Par nom de processus
pkill -f "http.server 8080"
pkill -f "http.server 8081"
pkill -f "http.server 8082"

# Méthode 2: Trouver les PIDs puis tuer
ps aux | grep "http.server" | grep -v grep
# Puis: kill <PID>

# Méthode 3: Tuer tous les serveurs http.server
pkill -f "python3 -m http.server"
```

### 🗑️ Désactiver le Forward Proxy

```bash
# Supprimer la configuration
rm -f /etc/nginx/conf.d/forward_proxy.conf

# Tester et recharger
nginx -t && systemctl reload nginx
```

---

## 🔧 Dépannage

### ❌ Problème : "Address already in use"

```bash
# Vérifier quel processus utilise le port
sudo lsof -i :80
sudo lsof -i :8080

# Tuer le processus
kill <PID>
```

### ❌ Problème : "502 Bad Gateway"

**Causes possibles :**

1. Le backend n'est pas démarré
2. Le backend est sur le mauvais port
3. Firewall bloque la connexion

**Solution :**

```bash
# Vérifier que les backends tournent
ss -tlnp | grep -E '8080|8081|8082'

# Redémarrer les backends si nécessaire
pkill -f "http.server 8080"
nohup python3 -m http.server 8080 --directory /var/www/backend_apache > /var/log/backend8080.log 2>&1 &

# Vérifier les logs Nginx
tail -50 /var/log/nginx/error.log
```

### ❌ Problème : "nginx: configuration file /etc/nginx/nginx.conf test failed"

```bash
# Voir le détail de l'erreur
nginx -t

# Erreurs communes:
# - Accolade manquante
# - Point-virgule manquant
# - Directive inconnue
# - Fichier include introuvable
```

### ❌ Problème : Le load balancer ne distribue pas les requêtes

```bash
# Vérifier que les deux backends répondent
curl http://127.0.0.1:8081/
curl http://127.0.0.1:8082/

# Vérifier la configuration upstream
grep -A5 "upstream backend_pool" /etc/nginx/sites-available/nginx_site

# Forcer plusieurs requêtes
for i in {1..10}; do curl -s http://127.0.0.1/lb/ | grep -o "Backend [0-9]"; done
```

### ❌ Problème : Forward proxy refuse la connexion

```bash
# Vérifier que le port 8888 écoute
ss -tlnp | grep 8888

# Tester depuis localhost
curl -v -x http://127.0.0.1:8888 http://example.com

# Vérifier les règles allow/deny
grep -A10 "listen 8888" /etc/nginx/conf.d/forward_proxy.conf
```

---

## 📚 Concepts clés

### 🔄 Round-Robin

Algorithme de distribution de charge qui alterne entre les backends dans l'ordre :

- Simple à implémenter
- Distribution équitable (si les backends ont la même capacité)
- Ne tient pas compte de la charge réelle de chaque backend

**Autres algorithmes possibles :**

- `least_conn` : envoie vers le backend avec le moins de connexions actives
- `ip_hash` : même client → toujours le même backend (sticky sessions)
- `weight` : pondération des backends (ex: backend1 reçoit 70% du trafic)

### 🔒 Headers proxy

Les headers ajoutés par Nginx permettent au backend de connaître le client réel :

```
X-Real-IP: 192.168.1.100          ← IP du client original
X-Forwarded-For: 192.168.1.100    ← IP du client (peut être une liste)
X-Forwarded-Proto: https          ← Protocole utilisé (http/https)
Host: example.com                  ← Nom de domaine demandé
```

### ⏱️ Timeouts

```
proxy_connect_timeout 5s;   ← Temps max pour établir connexion avec backend
proxy_read_timeout 30s;     ← Temps max pour lire réponse du backend
proxy_send_timeout 30s;     ← Temps max pour envoyer requête au backend
```

### 🔄 Failover automatique

```nginx
proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
```

Si un backend retourne une erreur, Nginx essaie automatiquement le backend suivant.

---

## 📊 Résumé de l'architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              SERVEUR VM                                  │
│                                                                           │
│  Port 80 (Nginx)                                                         │
│  ├─ /              → /var/www/nginx_site/        (Site statique)        │
│  ├─ /apache/       → 127.0.0.1:8080              (Reverse proxy)        │
│  └─ /lb/           → 8081 + 8082 (round-robin)   (Load balancer)        │
│                                                                           │
│  Port 8888 (Nginx)                                                       │
│  └─ /              → Internet                     (Forward proxy)        │
│                                                                           │
│  Backends Python                                                         │
│  ├─ 8080 → /var/www/backend_apache/                                     │
│  ├─ 8081 → /var/www/backend1/                                           │
│  └─ 8082 → /var/www/backend2/                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Checklist finale

- [ ] Les 3 backends Python sont démarrés (ports 8080, 8081, 8082)
- [ ] Nginx écoute sur le port 80
- [ ] Le site statique répond sur `http://localhost/`
- [ ] Le reverse proxy fonctionne sur `http://localhost/apache/`
- [ ] Le load balancer alterne sur `http://localhost/lb/`
- [ ] Le forward proxy est accessible sur le port 8888
- [ ] Les logs sont consultables dans `/var/log/`

---

## 📝 Notes importantes

1. **Python HTTP Server** : Simple pour le TP mais **ne pas utiliser en production** (pas sécurisé, pas performant)

2. **Forward Proxy** : Configuration minimale, pour un usage production, considérer :

   - Squid (proxy HTTP/HTTPS complet)
   - Authentification
   - Filtrage de contenu
   - Cache

3. **HTTPS** : Cette configuration est en HTTP. Pour HTTPS :

   - Générer certificats SSL (Let's Encrypt)
   - Configurer `listen 443 ssl`
   - Ajouter `ssl_certificate` et `ssl_certificate_key`

4. **Sécurité** :
   - Changer `server_name _` par votre domaine
   - Activer le firewall (ufw/iptables)
   - Limiter les connexions (`limit_req_zone`)
   - Ajouter headers de sécurité (HSTS, CSP, etc.)

---

## 🎓 Pour aller plus loin

### Load Balancing avancé

- Sticky sessions avec `ip_hash`
- Health checks avec `max_fails` et `fail_timeout`
- Backup servers
- Weights (pondération)

### Cache

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m;
proxy_cache my_cache;
proxy_cache_valid 200 60m;
```

### Compression

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
```

### Rate Limiting

```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
limit_req zone=mylimit burst=20;
```

---

**Auteur :** Configuration pour TP réseau  
**Date :** Novembre 2025  
**Version :** 1.0
