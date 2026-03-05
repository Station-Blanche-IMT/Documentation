---
title: Installation & Déploiement
layout: default
parent: Application Web
nav_order: 3
---

# Installation & Déploiement
{: .no_toc }

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prérequis

- **OS :** Linux (Debian/Ubuntu recommandé)
- **Python :** 3.12 ou supérieur
- **ClamAV :** Paquet `clamav` installé (`clamscan` disponible dans le PATH)
- **Apache2** (pour la production)
- **Paquets Python :** voir `requirements.txt`

---

## Installation en développement

```bash
# 1. Cloner le dépôt
git clone https://github.com/Station-Blanche-ITM/station-blanche-web.git
cd station-blanche-web

# 2. Créer un environnement virtuel
python3 -m venv venv
source venv/bin/activate

# 3. Installer les dépendances
pip install -r requirements.txt

# 4. Appliquer les migrations
cd sb_proj
python3 manage.py migrate

# 5. Créer les utilisateurs Django obligatoires
python3 manage.py createsuperuser  # Utilisateur "admin"
# Puis dans le shell Django :
python3 manage.py shell -c "
from django.contrib.auth.models import User
User.objects.create_user('user', password='user_password_placeholder')
"

# 6. Lancer le serveur de développement
python3 manage.py runserver
```

> **Important :** Deux utilisateurs Django sont nécessaires :
> - `admin` (superuser) — utilisé pour l'accès administrateur via PIN
> - `user` (utilisateur standard) — utilisé pour la connexion via badge RFID
{: .warning }

---

## Déploiement en production

Le script `Start_App.sh` automatise tout le déploiement :

```bash
sudo bash Start_App.sh
```

Ce script :
1. Vérifie Python 3
2. Installe Apache2 si nécessaire
3. Crée/met à jour l'environnement virtuel `.venv`
4. Installe les dépendances pip
5. Collecte les fichiers statiques (`collectstatic`)
6. Applique les migrations
7. Déploie le service systemd Gunicorn
8. Configure Apache2 en reverse proxy
9. Applique les bonnes permissions sur la BDD et les logs
10. Redémarre les services

---

## Service systemd

Le fichier `station-blanche-web.service` configure Gunicorn comme service système :

- **Chemin du venv :** `/home/kiosk/station-blanche-web/.venv`
- **Répertoire de travail :** `/home/kiosk/station-blanche-web/sb_proj`
- **Port :** 8000 (interne uniquement)

---

## Configuration Apache

Le fichier `station-blanche-web.conf` configure le reverse proxy :

- Reverse proxy HTTP sur le port 80 → Gunicorn sur le port 8000
- Pages d'erreur personnalisées (`error-pages/error.html`)

---

## Permissions requises

L'utilisateur système (ex: `kiosk`) doit pouvoir exécuter :
- `sudo systemctl poweroff`
- `sudo systemctl reboot`
- Lecture/écriture sur `/mnt/fileshare/`
- Lecture des périphériques USB (`lsblk`, montage)
- Lecture/écriture sur `/var/lib/station-blanche/staging/` (répertoire de staging pour les scans)

Fichiers nécessitant des permissions spécifiques :
- La BDD SQLite : `chmod 664 db.sqlite3`
- Le dossier de logs : `chmod -R 775 sb_app/logs/`
