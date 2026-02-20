# Cloud-1 — Déploiement automatisé WordPress sur le Cloud

---

## PARTIE 1 — Guide de déploiement complet

### Prérequis

- Un nom de domaine (OVH, Gandi, Namecheap...)
- Une VM cloud (AWS, OVH, DigitalOcean, Hetzner...) sous Ubuntu 22.04+
- Une clé SSH pour accéder à la VM
- Git et Homebrew installés en local (macOS)

---

### Étape 1 — Créer la VM sur AWS (EC2)

1. Aller sur **AWS Console → EC2 → Launch Instance**
2. Choisir **Ubuntu Server 22.04 LTS**
3. Type d'instance : `t2.micro` (free tier) ou `t3.small` selon les besoins
4. Créer ou importer une clé SSH → télécharger le fichier `.pem`
5. Dans **Security Group**, ouvrir les ports :
   - `22` (SSH)
   - `80` (HTTP)
   - `443` (HTTPS)
6. Lancer l'instance et noter l'**IP publique**

Placer la clé SSH dans le bon dossier et sécuriser ses permissions :

```bash
mv ~/Downloads/cloud1-key.pem ~/.ssh/cloud1-key.pem
chmod 400 ~/.ssh/cloud1-key.pem
```

---

### Étape 2 — Configurer le DNS

Dans le panneau de gestion de ton registrar (OVH, Gandi...),
ajouter deux enregistrements de type **A** :

| Sous-domaine | Type | Cible          |
|--------------|------|----------------|
| @            | A    | IP_DU_SERVEUR  |
| www          | A    | IP_DU_SERVEUR  |

> La propagation DNS peut prendre de quelques minutes à quelques heures.
> Pour tester : `dig +short ton-domaine.fr @8.8.8.8`
> Quand la commande retourne l'IP du serveur, le DNS est propagé.

---

### Étape 3 — Configurer le projet en local

Cloner le projet :

```bash
git clone <url-du-repo> cloud-1
cd cloud-1
```

Créer le fichier `.env` à la racine avec les credentials de la base de données :

```bash
cat > .env << EOF
MYSQL_DATABASE=wordpress
MYSQL_USER=wp_user
MYSQL_PASSWORD=un_mot_de_passe_solide
MYSQL_ROOT_PASSWORD=un_root_password_solide

DOMAIN=ton-domaine.fr
LETSENCRYPT_EMAIL=ton@email.com
EOF
```

> Ne jamais commiter le fichier `.env` (il est dans le `.gitignore`).

Mettre à jour le fichier `nginx/default.conf` avec ton nom de domaine :

```nginx
server_name ton-domaine.fr www.ton-domaine.fr;
```

Et les chemins des certificats :

```nginx
ssl_certificate     /etc/letsencrypt/live/ton-domaine.fr/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/ton-domaine.fr/privkey.pem;
```

Mettre à jour l'inventaire Ansible `ansible/inventory` avec l'IP du serveur :

```ini
[cloud1]
IP_DU_SERVEUR ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/cloud1-key.pem
```

---

### Étape 4 — Installer Ansible en local

```bash
brew install ansible
```

Ajouter le serveur aux known hosts SSH :

```bash
ssh-keyscan -H IP_DU_SERVEUR >> ~/.ssh/known_hosts
```

---

### Étape 5 — Déployer avec Ansible

```bash
cd ansible
ansible-playbook -i inventory site.yml
```

Le playbook va automatiquement :
- Mettre à jour `apt`
- Installer Docker et Docker Compose
- Copier les fichiers du projet sur le serveur
- Lancer la stack avec `docker compose up -d`

---

### Étape 6 — Générer le certificat SSL (Let's Encrypt)

Le certificat doit être généré **avant** de lancer nginx en mode HTTPS.
Se connecter au serveur et lancer Certbot :

```bash
# Installer Certbot
sudo snap install --classic certbot

# Stopper nginx pour libérer le port 80
sudo docker stop cloud1-nginx

# Générer le certificat
sudo certbot certonly --standalone \
  -d ton-domaine.fr \
  -d www.ton-domaine.fr \
  --non-interactive \
  --agree-tos \
  --email ton@email.com

# Redémarrer la stack complète
sudo docker compose -f /opt/cloud-1/docker-compose.yml up -d
```

> Le certificat est valable 90 jours. Certbot installe automatiquement
> un timer systemd pour le renouvellement.

---

### Étape 7 — Installer WordPress

Ouvrir `https://ton-domaine.fr` dans un navigateur.
Suivre l'assistant d'installation :

1. Choisir la langue
2. Renseigner :
   - Titre du site
   - Nom d'utilisateur admin
   - Mot de passe admin (le conserver précieusement)
   - Adresse email
3. Cliquer **Installer WordPress**
4. Se connecter via `https://ton-domaine.fr/wp-admin`

---

### Commandes utiles

```bash
# Voir l'état des containers
ssh -i ~/.ssh/cloud1-key.pem ubuntu@IP_DU_SERVEUR \
  "sudo docker compose -f /opt/cloud-1/docker-compose.yml ps"

# Voir les logs d'un service
ssh -i ~/.ssh/cloud1-key.pem ubuntu@IP_DU_SERVEUR \
  "sudo docker logs cloud1-nginx --tail=50"

# Redémarrer un service
ssh -i ~/.ssh/cloud1-cert.pem ubuntu@IP_DU_SERVEUR \
  "sudo docker restart cloud1-nginx"

# Vérifier l'espace disque
ssh -i ~/.ssh/cloud1-key.pem ubuntu@IP_DU_SERVEUR "df -h /"

# Nettoyer Docker si le disque est plein
ssh -i ~/.ssh/cloud1-key.pem ubuntu@IP_DU_SERVEUR \
  "sudo docker system prune -af"

# Vérifier la propagation DNS
dig +short ton-domaine.fr @8.8.8.8

# Renouveler le certificat manuellement
ssh -i ~/.ssh/cloud1-key.pem ubuntu@IP_DU_SERVEUR \
  "sudo certbot renew --quiet"
```

---

---

## PARTIE 2 — Explication du projet pour la correction

### Vue d'ensemble

Cloud-1 est un projet **42** dont l'objectif est de déployer une infrastructure WordPress
complète sur un serveur cloud distant, de manière **entièrement automatisée**, sécurisée
et persistante. Le déploiement s'inspire du projet Inception mais dans un contexte cloud réel.

---

### Architecture

```
Internet
    │
    │ HTTPS (443) / HTTP (80)
    ▼
┌─────────────┐
│    Nginx    │  ← Reverse proxy, point d'entrée unique
│  (port 80)  │    Gère le SSL/TLS (Let's Encrypt)
│  (port 443) │    Redirige HTTP → HTTPS
└──────┬──────┘
       │
       ├──────────────────────────┐
       │                          │
       ▼                          ▼
┌─────────────┐          ┌──────────────────┐
│  WordPress  │          │   phpMyAdmin     │
│  PHP-FPM    │          │   (port 80)      │
│  (port 9000)│          └────────┬─────────┘
└──────┬──────┘                   │
       │                          │
       └──────────┬───────────────┘
                  ▼
          ┌──────────────┐
          │   MariaDB    │
          │  (port 3306) │
          └──────────────┘
```

---

### Les services Docker

#### MariaDB (`cloud1-db`)
- Base de données SQL qui stocke tout le contenu WordPress
- Uniquement sur le réseau `backend` — jamais exposée à Internet
- Les données sont persistées dans le volume `db_data`
- Configurée via variables d'environnement (`.env`)

#### WordPress (`cloud1-wordpress`)
- Tourne en mode **PHP-FPM** (FastCGI Process Manager)
- PHP-FPM ne sert pas les fichiers statiques lui-même : il exécute uniquement le PHP
- Nginx reçoit les requêtes et délègue l'exécution PHP à WordPress via FastCGI sur le port 9000
- Les fichiers WordPress sont stockés dans le volume `wp_data`

#### Nginx (`cloud1-nginx`)
- **Reverse proxy** : point d'entrée unique de toute l'infrastructure
- Rôles :
  - Redirection automatique HTTP → HTTPS (301)
  - Terminaison SSL/TLS avec les certificats Let's Encrypt
  - Transmission des requêtes PHP à WordPress via FastCGI
  - Proxy des requêtes `/phpmyadmin` vers le container phpMyAdmin
- Partage le volume `wp_data` en lecture seule pour servir les fichiers statiques (images, CSS, JS)

#### phpMyAdmin (`cloud1-phpmyadmin`)
- Interface graphique pour administrer la base de données
- Accessible via `https://ton-domaine.fr/phpmyadmin`
- Sur les deux réseaux : `backend` pour accéder à MariaDB, `frontend` pour être proxyfié par Nginx

---

### Les réseaux Docker

Deux réseaux isolés sont utilisés pour cloisonner les services :

| Réseau     | Services connectés                        | Rôle                              |
|------------|-------------------------------------------|-----------------------------------|
| `frontend` | Nginx, phpMyAdmin                         | Réseau exposé, accessible depuis l'extérieur via Nginx |
| `backend`  | Nginx, WordPress, MariaDB, phpMyAdmin     | Réseau interne, communication entre services |

MariaDB n'est **que** sur le réseau `backend` : elle est totalement inaccessible depuis l'extérieur.

---

### La persistance des données

Deux volumes Docker nommés garantissent que les données survivent aux redémarrages :

| Volume    | Monté dans              | Contenu                                |
|-----------|-------------------------|----------------------------------------|
| `db_data` | `/var/lib/mysql`        | Données MariaDB (tables, utilisateurs) |
| `wp_data` | `/var/www/html`         | Fichiers WordPress (thèmes, plugins, uploads) |

Ces données persistent après :
- `docker compose restart`
- `docker compose down` puis `up`
- Reboot du serveur

---

### La sécurité

- **Seuls les ports 22, 80 et 443** sont exposés sur le serveur (via le Security Group AWS)
- MariaDB n'est **jamais accessible** depuis Internet
- Le trafic HTTP est **automatiquement redirigé** en HTTPS (301 permanent)
- Le SSL/TLS utilise **TLSv1.2 et TLSv1.3** uniquement (les versions obsolètes sont désactivées)
- Les certificats sont fournis par **Let's Encrypt** (autorité de certification gratuite et reconnue)
- Le renouvellement des certificats est **automatique** via Certbot
- Les credentials de la base ne sont jamais dans le code — ils sont dans un fichier `.env` non commité

---

### L'automatisation avec Ansible

Le playbook `ansible/site.yml` permet de déployer l'infrastructure sur **n'importe quel serveur Ubuntu** avec une seule commande.

Étapes automatisées :

1. `apt update` — mise à jour des paquets
2. Installation des dépendances système (`curl`, `ca-certificates`, `gnupg`)
3. Installation de **Docker** via le script officiel
4. Installation du plugin **Docker Compose**
5. Création du répertoire `/opt/cloud-1`
6. Copie des fichiers du projet via **rsync** (en excluant `.git` et les venvs)
7. Lancement de la stack avec `docker compose up -d`

L'inventaire (`ansible/inventory`) définit l'hôte cible, l'utilisateur SSH et la clé privée.

---

### Le certificat SSL — Processus complet

1. Let's Encrypt vérifie que le serveur contrôle bien le domaine via un **challenge HTTP-01**
2. Certbot démarre un mini serveur web temporaire sur le port 80
3. Let's Encrypt appelle `http://ton-domaine.fr/.well-known/acme-challenge/<token>`
4. Si la réponse est correcte, le certificat est émis
5. Les fichiers `fullchain.pem` et `privkey.pem` sont stockés dans `/etc/letsencrypt/live/`
6. Ce dossier est monté en lecture seule dans le container Nginx

---

### Structure des fichiers

```
cloud-1/
├── .env                        # Variables d'environnement (non commité)
├── docker-compose.yml          # Définition de la stack Docker
├── nginx/
│   └── default.conf            # Configuration Nginx (reverse proxy + SSL)
├── ansible/
│   ├── inventory               # Adresse IP et accès SSH du serveur
│   └── site.yml                # Playbook de déploiement automatisé
└── README.md                   # Ce fichier
```

---

### Pourquoi PHP-FPM et pas Apache ?

WordPress peut tourner avec Apache (qui intègre PHP directement) ou avec **Nginx + PHP-FPM**.
Ce projet utilise Nginx + PHP-FPM car :

- **Séparation des responsabilités** : Nginx gère le HTTP, PHP-FPM gère l'exécution PHP
- **Performance** : PHP-FPM en mode pool de processus est plus efficace qu'Apache pour les charges importantes
- **Architecture microservices** : chaque container a une seule responsabilité (principe Docker)
- **Flexibilité** : Nginx peut servir les fichiers statiques directement sans passer par PHP

---

### Différence avec Inception

| Inception | Cloud-1 |
|-----------|---------|
| VM locale (VirtualBox) | Serveur cloud réel (AWS, OVH...) |
| Images Docker custom (Dockerfile) | Images Docker officielles |
| Déploiement manuel | Déploiement automatisé via Ansible |
| SSL auto-signé | SSL Let's Encrypt (certificat reconnu) |
| Accès local uniquement | Accessible depuis Internet |
