# 🌐 ATELIER : INTERCONNEXION RÉSEAU MULTISITES - GUIDE DÉBUTANT

## 📋 Table des matières

- [Introduction](#introduction)
- [Étape 0 : Préparation de Packet Tracer](#étape-0--préparation-de-packet-tracer)
- [Étape 1 : Construction de la topologie](#étape-1--construction-de-la-topologie)
- [Étape 2 : Configuration de base des équipements](#étape-2--configuration-de-base-des-équipements)
- [Étape 3 : Configuration RIP (Site A)](#étape-3--configuration-rip-site-a)
- [Étape 4 : Configuration OSPF (Site B)](#étape-4--configuration-ospf-site-b)
- [Étape 5 : Configuration BGP (Interconnexion)](#étape-5--configuration-bgp-interconnexion)
- [Étape 6 : Tests et vérifications](#étape-6--tests-et-vérifications)
- [Étape 7 : Configuration Haute Disponibilité (HSRP)](#étape-7--configuration-haute-disponibilité-hsrp)
- [Étape 8 : Sécurisation (ACL et Authentification)](#étape-8--sécurisation-acl-et-authentification)

---

## 🎯 Introduction

**Objectifs de l'atelier :**

- Apprendre à interconnecter plusieurs sites d'entreprise
- Configurer 3 protocoles de routage différents (RIP, OSPF, BGP)
- Tester la communication entre sites
- Ajouter de la haute disponibilité et de la sécurité

**Durée estimée :** 2 heures

**Ce que vous allez créer :**

```
Site A (RIP)                          Site B (OSPF)
AS 65001                              AS 65002

  PC1                                   PC3
   |                                     |
[Switch1]                            [Switch3]
   |                                     |
  [R1]----------(RIP)----------        [R3]
   |                           |         |
  [R2]----------(BGP)--------[R4]    [Switch4]
   |                                     |
[Switch2]                               PC4
   |
  PC2
```

---

## 📦 Étape 0 : Préparation de Packet Tracer

### 0.1 Ouvrir Packet Tracer

1. Lancez Cisco Packet Tracer
2. Cliquez sur **File** → **New** (ou Ctrl+N)
3. Vous avez maintenant un espace de travail vide

### 0.2 Familiarisation avec l'interface

- **Bas de l'écran** : Barre d'outils avec les équipements
- **Zone centrale** : Espace de travail pour placer les équipements
- **Bas gauche** : Types d'équipements (Routers, Switches, End Devices, etc.)

---

## 🏗️ Étape 1 : Construction de la topologie

### 1.1 Ajouter les routeurs

**Comment ajouter un routeur :**

1. Cliquez sur l'icône **Routers** (en bas à gauche)
2. Sélectionnez **1841** ou **2911** (routeurs Cisco)
3. Cliquez 4 fois dans l'espace de travail pour placer R1, R2, R3, R4

**Renommer les routeurs :**

1. Cliquez sur un routeur
2. En haut de la fenêtre qui s'ouvre, changez le nom (ex: "R1")
3. Répétez pour R2, R3, R4

### 1.2 Ajouter les switches

1. Cliquez sur **Switches** (en bas)
2. Sélectionnez **2960** (switch standard)
3. Placez 4 switches dans l'espace de travail
4. Renommez-les : Switch1, Switch2, Switch3, Switch4

### 1.3 Ajouter les PCs

1. Cliquez sur **End Devices** (en bas)
2. Sélectionnez **PC**
3. Placez 4 PCs
4. Renommez-les : PC1, PC2, PC3, PC4

### 1.4 Connecter les équipements

**Types de câbles à utiliser :**

- **Câble droit (Copper Straight-Through)** : pour routeur ↔ switch et switch ↔ PC
- **Câble série (Serial DCE)** : pour routeur ↔ routeur

**Connexions à réaliser :**

#### Site A :

1. Cliquez sur l'icône **Câble** (éclair orange)
2. Sélectionnez **Copper Straight-Through**
3. **PC1** → **Switch1** (FastEthernet0 des deux côtés)
4. **Switch1** → **R1** (FastEthernet du switch → GigabitEthernet0/0 du routeur)
5. **PC2** → **Switch2** → **R2**

#### Liaison entre R1 et R2 :

1. Sélectionnez **Serial DCE**
2. **R1 Serial0/0/0** → **R2 Serial0/0/0**

#### Site B :

1. **PC3** → **Switch3** → **R3**
2. **PC4** → **Switch4** → **R4**
3. **R3 Serial0/0/0** → **R4 Serial0/0/0** (Serial DCE)

#### Liaison intersite (BGP) :

1. **R2 Serial0/0/1** → **R3 Serial0/0/1** (Serial DCE)

**🔍 Vérification :** Vous devez avoir une topologie complète avec :

- 4 routeurs
- 4 switches
- 4 PCs
- Tous connectés selon le schéma

---

## ⚙️ Étape 2 : Configuration de base des équipements

### 2.1 Plan d'adressage IP

Voici le plan d'adressage que nous allons utiliser :

| Équipement | Interface     | Adresse IP   | Masque          | Réseau            |
| ---------- | ------------- | ------------ | --------------- | ----------------- |
| **Site A** |
| R1         | Gi0/0         | 192.168.1.1  | 255.255.255.0   | LAN1              |
| R1         | Se0/0/0       | 10.1.1.1     | 255.255.255.252 | Liaison R1-R2     |
| R2         | Gi0/0         | 192.168.2.1  | 255.255.255.0   | LAN2              |
| R2         | Se0/0/0       | 10.1.1.2     | 255.255.255.252 | Liaison R1-R2     |
| R2         | Se0/0/1       | 10.2.2.1     | 255.255.255.252 | Liaison BGP R2-R3 |
| PC1        | FastEthernet0 | 192.168.1.10 | 255.255.255.0   | LAN1              |
| PC2        | FastEthernet0 | 192.168.2.10 | 255.255.255.0   | LAN2              |
| **Site B** |
| R3         | Se0/0/1       | 10.2.2.2     | 255.255.255.252 | Liaison BGP R2-R3 |
| R3         | Gi0/0         | 192.168.3.1  | 255.255.255.0   | LAN3              |
| R3         | Se0/0/0       | 10.3.3.1     | 255.255.255.252 | Liaison R3-R4     |
| R4         | Se0/0/0       | 10.3.3.2     | 255.255.255.252 | Liaison R3-R4     |
| R4         | Gi0/0         | 192.168.4.1  | 255.255.255.0   | LAN4              |
| PC3        | FastEthernet0 | 192.168.3.10 | 255.255.255.0   | LAN3              |
| PC4        | FastEthernet0 | 192.168.4.10 | 255.255.255.0   | LAN4              |

### 2.2 Configurer les PCs

**Pour chaque PC (exemple avec PC1) :**

1. Cliquez sur **PC1**
2. Allez dans l'onglet **Desktop**
3. Cliquez sur **IP Configuration**
4. Configurez :
   - **IP Address** : 192.168.1.10
   - **Subnet Mask** : 255.255.255.0
   - **Default Gateway** : 192.168.1.1 (l'IP du routeur R1)

**Répétez pour les autres PCs :**

- **PC2** : IP=192.168.2.10, Gateway=192.168.2.1
- **PC3** : IP=192.168.3.10, Gateway=192.168.3.1
- **PC4** : IP=192.168.4.10, Gateway=192.168.4.1

### 2.3 Configurer R1 (Routeur 1 - Site A)

**Accéder à la CLI du routeur :**

1. Cliquez sur **R1**
2. Allez dans l'onglet **CLI** (ligne de commande)
3. Appuyez sur **Entrée** si nécessaire

**Commandes à taper :**

```cisco
enable
configure terminal
hostname R1

! Configuration de l'interface LAN (vers Switch1 et PC1)
interface GigabitEthernet0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
exit

! Configuration de l'interface série vers R2
interface Serial0/0/0
ip address 10.1.1.1 255.255.255.252
clock rate 64000
no shutdown
exit

! Sauvegarder la configuration
end
write memory
```

**💡 Explications :**

- `enable` : passe en mode privilégié
- `configure terminal` : entre en mode configuration
- `hostname R1` : donne le nom "R1" au routeur
- `interface GigabitEthernet0/0` : sélectionne l'interface
- `ip address` : assigne l'adresse IP
- `no shutdown` : active l'interface
- `clock rate 64000` : définit la vitesse de l'horloge (nécessaire sur le côté DCE)
- `write memory` : sauvegarde la configuration

### 2.4 Configurer R2 (Routeur 2 - Site A)

```cisco
enable
configure terminal
hostname R2

! Interface LAN
interface GigabitEthernet0/0
ip address 192.168.2.1 255.255.255.0
no shutdown
exit

! Interface série vers R1
interface Serial0/0/0
ip address 10.1.1.2 255.255.255.252
no shutdown
exit

! Interface série vers R3 (liaison BGP)
interface Serial0/0/1
ip address 10.2.2.1 255.255.255.252
clock rate 64000
no shutdown
exit

end
write memory
```

### 2.5 Configurer R3 (Routeur 3 - Site B)

```cisco
enable
configure terminal
hostname R3

! Interface série vers R2 (liaison BGP)
interface Serial0/0/1
ip address 10.2.2.2 255.255.255.252
no shutdown
exit

! Interface LAN
interface GigabitEthernet0/0
ip address 192.168.3.1 255.255.255.0
no shutdown
exit

! Interface série vers R4
interface Serial0/0/0
ip address 10.3.3.1 255.255.255.252
clock rate 64000
no shutdown
exit

end
write memory
```

### 2.6 Configurer R4 (Routeur 4 - Site B)

```cisco
enable
configure terminal
hostname R4

! Interface série vers R3
interface Serial0/0/0
ip address 10.3.3.2 255.255.255.252
no shutdown
exit

! Interface LAN
interface GigabitEthernet0/0
ip address 192.168.4.1 255.255.255.0
no shutdown
exit

end
write memory
```

### 2.7 Vérification de la configuration de base

**Sur chaque routeur, vérifiez :**

```cisco
show ip interface brief
```

**Vous devez voir :**

- Toutes les interfaces configurées avec leur IP
- Le status "up" et protocol "up" pour chaque interface

**Testez la connectivité locale :**

Sur **R1** :

```cisco
ping 10.1.1.2
```

(doit répondre - c'est R2)

Sur **PC1** :

- Desktop → Command Prompt

```
ping 192.168.1.1
```

(doit répondre - c'est R1)

✅ **Si tous les pings fonctionnent, passez à l'étape suivante !**

---

## 🔄 Étape 3 : Configuration RIP (Site A)

**RIP (Routing Information Protocol)** est un protocole de routage simple qui compte le nombre de sauts (hops).

### 3.1 Configurer RIP sur R1

```cisco
enable
configure terminal

! Activation de RIP version 2
router rip
version 2
no auto-summary

! Annonce des réseaux connectés
network 192.168.1.0
network 10.0.0.0

end
write memory
```

**💡 Explications :**

- `router rip` : active le protocole RIP
- `version 2` : utilise RIPv2 (supporte VLSM et envoi du masque)
- `no auto-summary` : désactive le résumé automatique
- `network 192.168.1.0` : annonce le réseau LAN1
- `network 10.0.0.0` : annonce toutes les liaisons série en 10.x.x.x

### 3.2 Configurer RIP sur R2

```cisco
enable
configure terminal

router rip
version 2
no auto-summary

! Annonce des réseaux du Site A
network 192.168.2.0
network 10.0.0.0

end
write memory
```

### 3.3 Vérifier RIP

**Sur R1 et R2, tapez :**

```cisco
show ip route
```

**Vous devez voir :**

- Des routes marquées **R** (pour RIP)
- R1 doit connaître le réseau 192.168.2.0 (via R2)
- R2 doit connaître le réseau 192.168.1.0 (via R1)

**Vérifier les voisins RIP :**

```cisco
show ip rip database
```

**Tester la connectivité Site A :**

Sur **PC1** :

```
ping 192.168.2.10
```

(doit répondre - c'est PC2 sur le Site A via RIP)

✅ **Si le ping fonctionne, RIP est correctement configuré !**

---

## 🌐 Étape 4 : Configuration OSPF (Site B)

**OSPF (Open Shortest Path First)** est un protocole de routage à état de liens plus avancé que RIP.

### 4.1 Configurer OSPF sur R3

```cisco
enable
configure terminal

! Activation OSPF avec process ID 1
router ospf 1

! Annonce des réseaux dans l'area 0
network 192.168.3.0 0.0.0.255 area 0
network 10.3.3.0 0.0.0.3 area 0
network 10.2.2.0 0.0.0.3 area 0

end
write memory
```

**💡 Explications :**

- `router ospf 1` : active OSPF avec l'ID de processus 1
- `network [adresse] [wildcard mask] area 0` : annonce le réseau dans l'area 0
- **Wildcard mask** : inverse du masque de sous-réseau
  - 255.255.255.0 → wildcard = 0.0.0.255
  - 255.255.255.252 → wildcard = 0.0.0.3

### 4.2 Configurer OSPF sur R4

```cisco
enable
configure terminal

router ospf 1

! Annonce des réseaux du Site B
network 192.168.4.0 0.0.0.255 area 0
network 10.3.3.0 0.0.0.3 area 0

end
write memory
```

### 4.3 Vérifier OSPF

**Sur R3 et R4, tapez :**

```cisco
show ip route
```

**Vous devez voir :**

- Des routes marquées **O** (pour OSPF)
- R3 doit connaître le réseau 192.168.4.0 (via R4)
- R4 doit connaître le réseau 192.168.3.0 (via R3)

**Vérifier les voisins OSPF :**

```cisco
show ip ospf neighbor
```

Vous devez voir le voisin avec l'état **FULL**

**Tester la connectivité Site B :**

Sur **PC3** :

```
ping 192.168.4.10
```

(doit répondre - c'est PC4 sur le Site B via OSPF)

✅ **Si le ping fonctionne, OSPF est correctement configuré !**

---

## 🌍 Étape 5 : Configuration BGP (Interconnexion)

**BGP (Border Gateway Protocol)** est utilisé pour interconnecter différents systèmes autonomes (AS).

### 5.1 Comprendre les AS

- **Site A** = AS 65001 (système autonome 1)
- **Site B** = AS 65002 (système autonome 2)
- R2 et R3 sont les routeurs de bordure (border routers)

### 5.2 Configurer BGP sur R2

```cisco
enable
configure terminal

! Activation BGP avec l'AS 65001
router bgp 65001

! Déclaration du voisin BGP (R3)
neighbor 10.2.2.2 remote-as 65002

! Annonce des réseaux du Site A
network 192.168.1.0 mask 255.255.255.0
network 192.168.2.0 mask 255.255.255.0

end
write memory
```

**💡 Explications :**

- `router bgp 65001` : active BGP dans l'AS 65001
- `neighbor 10.2.2.2 remote-as 65002` : déclare R3 (10.2.2.2) comme voisin BGP dans l'AS 65002
- `network ... mask ...` : annonce les réseaux du Site A vers le Site B

### 5.3 Configurer BGP sur R3

```cisco
enable
configure terminal

! Activation BGP avec l'AS 65002
router bgp 65002

! Déclaration du voisin BGP (R2)
neighbor 10.2.2.1 remote-as 65001

! Annonce des réseaux du Site B
network 192.168.3.0 mask 255.255.255.0
network 192.168.4.0 mask 255.255.255.0

end
write memory
```

### 5.4 Redistribution des routes

Pour que RIP et OSPF connaissent les routes BGP, il faut redistribuer.

**Sur R2 (redistribuer BGP vers RIP) :**

```cisco
enable
configure terminal

! Dans RIP, redistribuer les routes BGP
router rip
redistribute bgp 65001 metric 1
exit

! Dans BGP, redistribuer les routes RIP
router bgp 65001
redistribute rip
exit

end
write memory
```

**Sur R3 (redistribuer BGP vers OSPF) :**

```cisco
enable
configure terminal

! Dans OSPF, redistribuer les routes BGP
router ospf 1
redistribute bgp 65002 subnets
exit

! Dans BGP, redistribuer les routes OSPF
router bgp 65002
redistribute ospf 1
exit

end
write memory
```

### 5.5 Vérifier BGP

**Sur R2 et R3, tapez :**

```cisco
show ip bgp summary
```

**Vous devez voir :**

- L'état du voisin BGP = **Established** (ou un nombre dans la colonne State/PfxRcd)

**Vérifier la table de routage :**

```cisco
show ip route
```

**Vous devez voir :**

- Des routes marquées **B** (pour BGP)
- R2 doit connaître les réseaux 192.168.3.0 et 192.168.4.0
- R3 doit connaître les réseaux 192.168.1.0 et 192.168.2.0

---

## ✅ Étape 6 : Tests et vérifications

### 6.1 Test de connectivité complète

**Test majeur : PC1 (Site A) vers PC3 (Site B)**

Sur **PC1** :

```
ping 192.168.3.10
```

✅ **Doit répondre !**

Sur **PC1** :

```
ping 192.168.4.10
```

✅ **Doit répondre !**

**Test inverse : PC3 (Site B) vers PC1 (Site A)**

Sur **PC3** :

```
ping 192.168.1.10
```

✅ **Doit répondre !**

### 6.2 Vérification des tables de routage

**Sur chaque routeur, vérifiez :**

```cisco
show ip route
```

**R1 doit connaître :**

- 192.168.1.0 (connecté)
- 192.168.2.0 (via RIP)
- 192.168.3.0 et 192.168.4.0 (via RIP redistribué de BGP)

**R4 doit connaître :**

- 192.168.4.0 (connecté)
- 192.168.3.0 (via OSPF)
- 192.168.1.0 et 192.168.2.0 (via OSPF redistribué de BGP)

### 6.3 Test de résilience

**Simuler une panne :**

1. Cliquez sur le câble entre **R2** et **R3**
2. Cliquez sur l'icône **Delete** (croix rouge)
3. Attendez 30 secondes

**Vérifiez :**

- Sur R2 : `show ip bgp summary` → le voisin n'est plus actif
- Sur R3 : les routes BGP disparaissent de `show ip route`
- Sur PC1 : `ping 192.168.3.10` → ne fonctionne plus

**Rétablir la liaison :**

1. Reconnectez R2 et R3 avec un câble série
2. Attendez la reconvergence (30-60 secondes)
3. Le ping doit refonctionner !

### 6.4 Commandes de diagnostic

**Tableau récapitulatif des commandes utiles :**

| Commande                  | Description                  |
| ------------------------- | ---------------------------- |
| `show ip interface brief` | État des interfaces          |
| `show ip route`           | Table de routage             |
| `show ip rip database`    | Base de données RIP          |
| `show ip ospf neighbor`   | Voisins OSPF                 |
| `show ip ospf interface`  | Interfaces OSPF              |
| `show ip bgp summary`     | État des sessions BGP        |
| `show ip bgp`             | Table BGP complète           |
| `ping [IP]`               | Test de connectivité         |
| `traceroute [IP]`         | Chemin suivi par les paquets |

---

## 🛡️ Étape 7 : Configuration Haute Disponibilité (HSRP)

**HSRP (Hot Standby Router Protocol)** permet d'avoir un routeur de secours automatique.

### 7.1 Principe

Nous allons configurer R1 et R2 pour partager une IP virtuelle **192.168.1.254** :

- Si R1 tombe en panne, R2 prend le relais automatiquement
- Les PCs utilisent l'IP virtuelle comme passerelle

### 7.2 Modification de la passerelle des PCs

**Sur PC1 :**

1. Desktop → IP Configuration
2. Changez **Default Gateway** : 192.168.1.254

### 7.3 Ajouter une interface à R2

Pour cet exemple, nous supposons que R2 a une interface supplémentaire sur le LAN du Site A.

**⚠️ Simplification :** Dans Packet Tracer, ajoutez une connexion supplémentaire entre R2 et Switch1 si possible, ou simulez mentalement.

### 7.4 Configurer HSRP sur R1

```cisco
enable
configure terminal

interface GigabitEthernet0/0
standby 1 ip 192.168.1.254
standby 1 priority 110
standby 1 preempt

end
write memory
```

**💡 Explications :**

- `standby 1 ip 192.168.1.254` : IP virtuelle partagée
- `standby 1 priority 110` : priorité de R1 (plus élevée = routeur actif)
- `standby 1 preempt` : permet à R1 de reprendre le rôle actif s'il revient après une panne

### 7.5 Configurer HSRP sur R2

```cisco
enable
configure terminal

interface GigabitEthernet0/0
standby 1 ip 192.168.1.254
standby 1 priority 100
standby 1 preempt

end
write memory
```

**Note :** R2 a une priorité plus faible (100), donc il est en standby.

### 7.6 Vérifier HSRP

```cisco
show standby
```

**Vous devez voir :**

- R1 = **Active**
- R2 = **Standby**

**Test de basculement :**

1. Désactivez l'interface de R1 : `interface Gi0/0` puis `shutdown`
2. R2 devient actif après quelques secondes
3. Le ping depuis PC1 continue de fonctionner !

---

## 🔒 Étape 8 : Sécurisation (ACL et Authentification)

### 8.1 Authentification OSPF (MD5)

Protège contre les annonces de routes frauduleuses.

**Sur R3 :**

```cisco
enable
configure terminal

! Sur l'interface connectée à R4
interface Serial0/0/0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 CISCO123
exit

end
write memory
```

**Sur R4 :**

```cisco
enable
configure terminal

interface Serial0/0/0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 CISCO123
exit

end
write memory
```

**Vérification :**

```cisco
show ip ospf neighbor
```

Les voisins doivent rester **FULL** avec authentification active.

### 8.2 Authentification BGP

**Sur R2 :**

```cisco
enable
configure terminal

router bgp 65001
neighbor 10.2.2.2 password BGP_SECRET
exit

end
write memory
```

**Sur R3 :**

```cisco
enable
configure terminal

router bgp 65002
neighbor 10.2.2.1 password BGP_SECRET
exit

end
write memory
```

**Vérification :**

```cisco
show ip bgp summary
```

La session BGP doit rester **Established**.

### 8.3 Configuration d'ACL (Access Control List)

**Exemple : Bloquer le trafic ICMP (ping) du Site A vers Site B**

**Sur R2 (avant l'interface vers R3) :**

```cisco
enable
configure terminal

! Créer une ACL étendue
access-list 100 deny icmp 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 100 deny icmp 192.168.2.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 100 permit ip any any

! Appliquer l'ACL sur l'interface sortante vers R3
interface Serial0/0/1
ip access-group 100 out
exit

end
write memory
```

**Test :**

- `ping 192.168.3.10` depuis PC1 → **ne fonctionne plus** (bloqué par ACL)
- Autre trafic → **fonctionne** (permit ip any any)

**Pour supprimer l'ACL :**

```cisco
configure terminal
interface Serial0/0/1
no ip access-group 100 out
exit
```

### 8.4 Sécurisation SSH (désactiver Telnet)

**Sur R1 (exemple) :**

```cisco
enable
configure terminal

! Configurer le nom de domaine
ip domain-name cisco.com

! Générer les clés RSA
crypto key generate rsa
! Choisissez 1024 bits

! Créer un utilisateur local
username admin privilege 15 secret Cisco123!

! Activer SSH sur les lignes VTY
line vty 0 4
transport input ssh
login local
exit

! Désactiver Telnet
no ip telnet server

end
write memory
```

**Test :**

- Depuis un autre routeur : `ssh -l admin 192.168.1.1`

### 8.5 Activation des logs de sécurité

**Sur tous les routeurs :**

```cisco
enable
configure terminal

! Activer les logs en mémoire tampon
logging buffered 16384

! Activer les logs de sécurité
logging trap debugging

! Activer les timestamps
service timestamps log datetime msec

end
write memory
```

**Voir les logs :**

```cisco
show logging
```

---

## 📊 Résumé de l'atelier

### ✅ Ce que vous avez appris :

1. **Construire une topologie réseau multisites** dans Packet Tracer
2. **Configurer les adresses IP** sur routeurs, switches et PCs
3. **Implémenter RIP** (protocole simple par comptage de sauts)
4. **Implémenter OSPF** (protocole à état de liens)
5. **Implémenter BGP** (pour interconnecter des AS différents)
6. **Redistribuer des routes** entre protocoles
7. **Configurer HSRP** pour la haute disponibilité
8. **Sécuriser le réseau** avec authentification et ACL
9. **Diagnostiquer et tester** la connectivité

### 📈 Points clés :

| Protocole | Usage                 | Avantages            | Inconvénients         |
| --------- | --------------------- | -------------------- | --------------------- |
| **RIP**   | Petits réseaux        | Simple, facile       | Lent, limite 15 sauts |
| **OSPF**  | Réseaux moyens/grands | Rapide, évolutif     | Plus complexe         |
| **BGP**   | Interconnexion AS     | Politique de routage | Très complexe         |

### 🎯 Compétences acquises :

- ✅ Interconnexion réseau multisites
- ✅ Configuration multi-protocoles
- ✅ Haute disponibilité avec HSRP
- ✅ Sécurisation réseau (ACL, authentification)
- ✅ Diagnostic et dépannage réseau

---

## 🔧 Dépannage (Troubleshooting)

### Problème : Les pings ne fonctionnent pas

**Vérifications à faire :**

1. **Interfaces actives ?**

   ```cisco
   show ip interface brief
   ```

   Toutes doivent être "up/up"

2. **Adresses IP correctes ?**

   ```cisco
   show running-config
   ```

3. **Protocoles de routage actifs ?**

   ```cisco
   show ip protocols
   ```

4. **Routes dans la table ?**

   ```cisco
   show ip route
   ```

5. **Passerelle par défaut sur PC ?**
   - Desktop → IP Configuration → vérifier Default Gateway

### Problème : BGP ne s'établit pas

```cisco
show ip bgp summary
```

- **State = Idle** → Problème de connectivité ou de configuration
- **State = Active** → Le routeur essaie de se connecter
- **State = Established** → ✅ OK

**Solutions :**

- Vérifier les adresses IP des voisins
- Vérifier les numéros d'AS
- Ping entre R2 et R3 doit fonctionner

### Problème : OSPF n'a pas de voisins

```cisco
show ip ospf neighbor
```

**Si vide :**

- Vérifier que les deux routeurs sont dans la même area
- Vérifier les masques wildcard
- Vérifier que les interfaces sont dans OSPF : `show ip ospf interface`

---

## 📚 Ressources supplémentaires

### Commandes essentielles Cisco

```cisco
enable                          # Mode privilégié
configure terminal              # Mode configuration
hostname [nom]                  # Nommer le routeur
interface [type] [numéro]       # Sélectionner une interface
ip address [IP] [masque]        # Configurer l'IP
no shutdown                     # Activer l'interface
exit                            # Sortir d'un niveau
end                             # Retour au mode privilégié
write memory                    # Sauvegarder (ou copy run start)
show running-config             # Voir la config active
show startup-config             # Voir la config sauvegardée
reload                          # Redémarrer le routeur
```

### Protocoles de routage - Aide-mémoire

**RIP :**

```cisco
router rip
version 2
no auto-summary
network [réseau]
```

**OSPF :**

```cisco
router ospf [process-id]
network [adresse] [wildcard] area [numéro]
```

**BGP :**

```cisco
router bgp [AS-number]
neighbor [IP-voisin] remote-as [AS-voisin]
network [réseau] mask [masque]
```

---

## 🎓 Pour aller plus loin

### Extensions possibles :

1. **Ajouter des VLANs** sur les switches
2. **Configurer NAT** sur les routeurs de bordure
3. **Implémenter QoS** (Quality of Service)
4. **Ajouter SNMP** pour la supervision
5. **Configurer un serveur DHCP** sur les routeurs
6. **Implémenter VPN** entre sites

### Scénarios de tests avancés :

- Simuler des pannes multiples
- Optimiser les métriques de routage
- Implémenter du load-balancing
- Configurer route-maps pour le filtrage BGP

---

## 📞 Conseils finaux

### ⚠️ Erreurs fréquentes à éviter :

1. **Oublier `no shutdown`** sur les interfaces → elles restent désactivées
2. **Mauvais type de câble** → utiliser Serial DCE entre routeurs
3. **Oublier de sauvegarder** → toujours faire `write memory`
4. **Mauvaise passerelle sur PC** → doit pointer vers le routeur local
5. **Oublier la redistribution** → les protocoles ne se parlent pas automatiquement

### ✨ Bonnes pratiques :

- ✅ Testez après chaque étape
- ✅ Sauvegardez régulièrement (`write memory`)
- ✅ Documentez vos configurations
- ✅ Utilisez des noms clairs pour les équipements
- ✅ Vérifiez toujours avec `show ip route`

---

## 🏆 Conclusion

**Félicitations !** 🎉

Vous avez réussi à :

- Créer une infrastructure réseau multisites complète
- Configurer 3 protocoles de routage différents
- Implémenter de la haute disponibilité
- Sécuriser votre réseau

**Vous êtes maintenant capable de :**

- Concevoir des architectures réseau d'entreprise
- Interconnecter des sites distants
- Choisir le bon protocole de routage selon le contexte
- Diagnostiquer et résoudre des problèmes réseau

---

**📝 Note importante :** Ce guide est conçu pour des débutants. Chaque commande est expliquée pour faciliter la compréhension. N'hésitez pas à expérimenter et à tester différentes configurations !

**Bon courage dans votre apprentissage réseau ! 🚀**

---

_Guide créé pour Packet Tracer - Atelier réseau multisites_  
_Version 1.0 - Novembre 2025_
