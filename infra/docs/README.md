# Projet Fil Rouge - Infrastructure Réseau B2

## 📋 Présentation du Projet

Infrastructure réseau complète avec segmentation VLAN, services web, base de données et monitoring, déployée sur VirtualBox avec automatisation Ansible.

**Étudiant :** [Ton Nom]  
**Formation :** B2 Informatique  
**Date :** Décembre 2024

---

## 🏗️ Architecture Réseau

### Schéma Global

                           INTERNET
                               |
                        [VM6-Firewall]
                     (4 interfaces réseau)
                               |
        ┌──────────────────────┼──────────────────────┐
        |                      |                      |
   VLAN Admin           VLAN Services              VLAN DMZ
   10.10.99.0/24        10.10.20.0/24          10.10.10.0/24
        |                      |                      |
    ┌───▼───┐          ┌───┬───┬───┐            ┌────▼────┐
    │  VM1  │          │VM3│VM4│VM5 │            │   VM2   │
    │ Admin │          │DNS│ DB│Mon │            │   Web   │
    └───────┘          └───┴───┴───┘            └─────────┘

### Architecture Détaillée

| VM | Hostname | IP | VLAN | Services | RAM | CPU | Disk |
|----|----------|---------|------|----------|-----|-----|------|
| VM1 | admin.lab.local | 10.10.99.10 | Admin | Ansible | 2GB | 2 | 20GB |
| VM2 | web.lab.local | 10.10.10.20 | DMZ | Nginx, Docker | 3GB | 2 | 30GB |
| VM3 | dns.lab.local | 10.10.20.30 | Services | Bind9 | 1GB | 1 | 10GB |
| VM4 | db.lab.local | 10.10.20.40 | Services | PostgreSQL | 2GB | 2 | 40GB |
| VM5 | monitoring.lab.local | 10.10.20.50 | Services | Prometheus, Grafana | 3GB | 2 | 30GB |
| VM6 | firewall.lab.local | 10.10.99.1 / .20.1 / .10.1 | Tous | iptables, routage | 1GB | 1 | 10GB |

**Total ressources :** 12GB RAM, 10 CPU, 140GB disque

---

## 🌐 Plan d'Adressage IP

### VLAN Admin (10.10.99.0/24)
- **Gateway :** 10.10.99.1 (Firewall)
- **VM1-Admin :** 10.10.99.10
- **Usage :** Administration, Ansible, Sauvegardes

### VLAN DMZ (10.10.10.0/24)
- **Gateway :** 10.10.10.1 (Firewall)
- **VM2-Web :** 10.10.10.20
- **Usage :** Services exposés (HTTP/HTTPS)

### VLAN Services (10.10.20.0/24)
- **Gateway :** 10.10.20.1 (Firewall)
- **VM3-DNS :** 10.10.20.30
- **VM4-Database :** 10.10.20.40
- **VM5-Monitoring :** 10.10.20.50
- **Usage :** Services internes critiques

---

## 🔥 Règles Firewall

### Politique Générale
- **INPUT :** DROP (par défaut)
- **FORWARD :** DROP (par défaut)
- **OUTPUT :** ACCEPT

### Règles Principales

NAT vers Internet : iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
Admin → Tous : iptables -A FORWARD -s 10.10.99.0/24 -j ACCEPT
DMZ → Internet : iptables -A FORWARD -s 10.10.10.0/24 -o enp0s3 -j ACCEPT
DMZ → Database : iptables -A FORWARD -s 10.10.10.0/24 -d 10.10.20.40 -p tcp --dport 5432 -j ACCEPT
DMZ → DNS : iptables -A FORWARD -s 10.10.10.0/24 -d 10.10.20.30 -p udp --dport 53 -j ACCEPT
Services ↔ Services : iptables -A FORWARD -s 10.10.20.0/24 -d 10.10.20.0/24 -j ACCEPT
Services → Internet : iptables -A FORWARD -s 10.10.20.0/24 -o enp0s3 -j ACCEPT
Monitoring → Tous : iptables -A FORWARD -s 10.10.20.50 -p tcp --dport 9100 -j ACCEPT

---

## 🛠️ Services Installés

### VM1-Admin
- **Ansible 2.12+** : Gestion centralisée de la configuration
- **Playbooks** : Déploiement automatique de tous les services
- **SSH** : Connexion sans mot de passe vers toutes les VMs

### VM2-Web (DMZ)
- **Nginx** : Serveur web HTTP
- **Docker + Docker Compose** : Conteneurisation d'applications
- **Node Exporter** : Métriques système pour Prometheus

### VM3-DNS (Services)
- **Bind9** : Serveur DNS autoritaire pour la zone lab.local
- **Résolution de noms** : Tous les hosts de l'infrastructure
- **Zones inverses** : Résolution IP → nom

### VM4-Database (Services)
- **PostgreSQL 16** : Base de données relationnelle
- **Base de données** : app_prod
- **Utilisateur** : appuser
- **Accès réseau** : Configuré pour DMZ et Admin

### VM5-Monitoring (Services)
- **Prometheus** : Collecte de métriques
- **Grafana** : Visualisation et dashboards
- **Node Exporter** : Installé sur toutes les VMs

### VM6-Firewall
- **iptables** : Filtrage de paquets et règles de sécurité
- **Routage inter-VLAN** : Communication contrôlée entre réseaux
- **NAT** : Accès Internet pour toutes les VMs

---

## 🚀 Installation

### Prérequis

- **VirtualBox** 6.1 ou supérieur
- **Ubuntu Server 22.04 ISO**
- **PC hôte** : 16GB RAM minimum, 200GB disque libre

### Étape 1 : Créer les Réseaux VirtualBox

**Linux/Mac :**
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet0 --ip 10.10.10.1 --netmask 255.255.255.0
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet1 --ip 10.10.20.1 --netmask 255.255.255.0
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet2 --ip 10.10.99.1 --netmask 255.255.255.0

**Windows :**
Créer via VirtualBox → Fichier → Gestionnaire de réseau hôte

### Étape 2 : Installer VM6-Firewall (En Premier)

1. Créer une VM Ubuntu Server 22.04
2. Configurer 4 interfaces réseau :
   - Adapter 1 : NAT
   - Adapter 2 : vboxnet0 (DMZ)
   - Adapter 3 : vboxnet1 (Services)
   - Adapter 4 : vboxnet2 (Admin)
3. Installer Ubuntu Server
4. Configurer netplan avec les 4 IPs
5. Activer le routage IP : echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
6. Configurer iptables avec le script firewall-rules.sh

### Étape 3 : Installer VM1-Admin

1. Créer une VM Ubuntu Server 22.04
2. 1 interface réseau : vboxnet2 (Admin)
3. IP : 10.10.99.10/24
4. Gateway : 10.10.99.1
5. Installer Ansible : sudo apt install -y ansible
6. Configurer les clés SSH

### Étape 4 : Cloner et Configurer les Autres VMs

**Cloner VM1-Admin 4 fois, puis configurer pour chaque VM :**

- **VM2-Web** : Réseau vboxnet0, IP 10.10.10.20, Hostname web.lab.local
- **VM3-DNS** : Réseau vboxnet1, IP 10.10.20.30, Hostname dns.lab.local
- **VM4-Database** : Réseau vboxnet1, IP 10.10.20.40, Hostname db.lab.local
- **VM5-Monitoring** : Réseau vboxnet1, IP 10.10.20.50, Hostname monitoring.lab.local

**Pour chaque clone :**
1. Régénérer les clés SSH : sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server
2. Changer hostname : sudo hostnamectl set-hostname <nom>.lab.local
3. Modifier netplan avec la nouvelle IP
4. Redémarrer : sudo reboot

### Étape 5 : Déploiement Automatique avec Ansible

cd ~/ansible-lab
ansible-playbook playbooks/00-deploy-all.yml

**Temps total de déploiement : 15-20 minutes**

---

## 🧪 Tests et Validation

### Script de Tests Automatique

./test-infrastructure.sh

### Tests Manuels

#### Test 1 : Connectivité Réseau
ping -c 2 web.lab.local
ping -c 2 db.lab.local
ping -c 2 monitoring.lab.local
ping -c 2 8.8.8.8

**Résultat attendu :** Toutes les résolutions et pings réussissent

#### Test 2 : DNS
dig web.lab.local
nslookup db.lab.local
dig -x 10.10.10.20

**Résultat attendu :** Résolution correcte des noms

#### Test 3 : Serveur Web
curl http://web.lab.local

**Résultat attendu :** Page HTML affichée avec "Infrastructure Lab"

#### Test 4 : PostgreSQL
PGPASSWORD=SecureP@ssw0rd2025 psql -h db.lab.local -U appuser -d app_prod -c "SELECT * FROM test_table;"

**Résultat attendu :** 3 lignes de données affichées

#### Test 5 : Monitoring
- **Prometheus :** http://10.10.20.50:9090
- **Grafana :** http://10.10.20.50:3000 (admin/admin)

**Résultat attendu :**
- 7 targets UP dans Prometheus (prometheus + 6 node_exporters)
- Dashboard Node Exporter affiche les 6 VMs

---

## 📊 Accès aux Services

| Service | URL | Identifiants |
|---------|-----|--------------|
| **Nginx** | http://10.10.10.20 | - |
| **Prometheus** | http://10.10.20.50:9090 | - |
| **Grafana** | http://10.10.20.50:3000 | admin / admin |
| **PostgreSQL** | db.lab.local:5432 | appuser / SecureP@ssw0rd2025 |

---

## 🔧 Troubleshooting

### Problème : VM ne peut pas accéder à Internet

**Diagnostic :**
ip route
ping 8.8.8.8

**Solution :**
ssh inoco@firewall.lab.local
sudo iptables -L FORWARD -n -v
# Vérifier que les règles FORWARD existent pour le VLAN concerné

### Problème : DNS ne résout pas les noms

**Diagnostic :**
cat /etc/resolv.conf
# Doit contenir : nameserver 10.10.20.30

**Solution :**
# Désactiver systemd-resolved
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 10.10.20.30" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf

### Problème : Ansible "Permission denied"

**Solution :**
# Reconfigurer les clés SSH
ssh-copy-id inoco@<IP_VM>
# Vérifier sudo sans mot de passe
ssh inoco@<IP_VM> "sudo whoami"
# Doit afficher "root" sans demander de mot de passe

### Problème : PostgreSQL refuse la connexion

**Diagnostic :**
nc -zv db.lab.local 5432

**Solution :**
# Vérifier pg_hba.conf
ssh inoco@db.lab.local
sudo nano /etc/postgresql/*/main/pg_hba.conf
# Ajouter : host all all 10.10.0.0/16 md5
sudo systemctl restart postgresql

---

## 📁 Structure du Projet

ansible-lab/
├── README.md
├── deploy.sh
├── test-infrastructure.sh
├── ansible.cfg
├── inventories/
│   └── hosts.yml
├── group_vars/
├── host_vars/
├── playbooks/
│   ├── 00-deploy-all.yml
│   ├── 01-update-all.yml
│   ├── 02-install-docker.yml
│   ├── 03-install-nginx.yml
│   ├── 04-install-bind9.yml
│   ├── 09-disable-systemd-resolved.yml
│   ├── 10-install-postgresql.yml
│   ├── 11-install-monitoring.yml
│   └── 12-install-node-exporter.yml
├── roles/
├── files/
└── templates/

---

## 🎓 Points Clés pour la Soutenance

### Architecture
✅ Segmentation réseau en 3 VLANs distincts
✅ Firewall centralisé avec règles strictes par VLAN
✅ DMZ pour isoler les services exposés sur Internet
✅ Principe de défense en profondeur

### Sécurité
✅ Principe du moindre privilège (règles iptables restrictives)
✅ Filtrage inter-VLAN (DROP par défaut)
✅ Base de données non exposée directement à Internet
✅ Services critiques isolés dans le VLAN Services

### Automatisation
✅ Déploiement complet via Ansible
✅ Infrastructure as Code (reproductible)
✅ Playbooks modulaires et réutilisables
✅ Tests automatisés

### Monitoring
✅ Supervision complète avec Prometheus
✅ Dashboards Grafana pour visualisation
✅ Métriques système sur toutes les VMs
✅ Alerting possible (évolution future)

---

## 📈 Améliorations Futures

- HTTPS avec certificats Let's Encrypt ou auto-signés
- Sauvegardes automatisées avec Restic
- AlertManager pour les alertes Prometheus
- Application web conteneurisée (WordPress, Nextcloud)
- CI/CD avec GitLab Runner
- Haute disponibilité pour les services critiques
- Logs centralisés avec ELK Stack

---

## 📚 Documentation Technique

### Commandes Utiles Ansible

# Test connectivité
ansible all -m ping

# Exécuter une commande sur toutes les VMs
ansible all -a "hostname"

# Redémarrer un service
ansible web.lab.local -b -a "systemctl restart nginx"

# Déployer un service spécifique
ansible-playbook playbooks/02-install-docker.yml

### Commandes Utiles PostgreSQL

# Connexion
psql -h db.lab.local -U appuser -d app_prod

# Backup
pg_dump -h db.lab.local -U appuser app_prod > backup.sql

# Restore
psql -h db.lab.local -U appuser -d app_prod < backup.sql

# Lister les connexions actives
SELECT * FROM pg_stat_activity;

### Commandes Utiles Prometheus

# Recharger la configuration
curl -X POST http://10.10.20.50:9090/-/reload

# Vérifier les targets
curl http://10.10.20.50:9090/api/v1/targets

# Query via API
curl 'http://10.10.20.50:9090/api/v1/query?query=up'

### Commandes Utiles Firewall

# Voir toutes les règles
sudo iptables -L -n -v

# Voir les règles NAT
sudo iptables -t nat -L -n -v

# Sauvegarder les règles
sudo iptables-save > /etc/iptables/rules.v4

# Recharger les règles
sudo /etc/firewall-rules.sh

---

## 🔒 Sécurité et Bonnes Pratiques

### Credentials par Défaut à Changer

- **Grafana** : admin / admin → À changer au premier login
- **PostgreSQL** : appuser / SecureP@ssw0rd2025 → À changer en production
- **SSH** : Clés SSH configurées, mot de passe désactivé pour plus de sécurité

### Recommandations de Sécurité

1. **Changer tous les mots de passe par défaut**
2. **Activer le fail2ban** sur toutes les VMs exposées
3. **Configurer des sauvegardes régulières** de la base de données
4. **Mettre en place HTTPS** avec des certificats valides
5. **Activer les logs centralisés** pour l'audit
6. **Configurer AlertManager** pour être notifié des incidents