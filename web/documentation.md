# Documentation complète — Station Blanche Web

> **Version :** Mars 2026
> **Public visé :** Prochaine équipe d'étudiants / tout développeur reprenant le projet

---

## Table des matières

1. [Présentation du projet](#1-présentation-du-projet)
2. [Architecture technique](#2-architecture-technique)
3. [Structure des fichiers](#3-structure-des-fichiers)
4. [Installation & déploiement](#4-installation--déploiement)
5. [Configuration](#5-configuration)
6. [Fonctionnalités détaillées](#6-fonctionnalités-détaillées)
   - 6.1 [Authentification & contrôle d'accès](#61-authentification--contrôle-daccès)
   - 6.2 [Gestion des clés USB — Scan antivirus](#62-gestion-des-clés-usb--scan-antivirus)
   - 6.3 [Copie de fichiers (USB & serveur de fichiers)](#63-copie-de-fichiers-usb--serveur-de-fichiers)
   - 6.4 [Nettoyage de clé USB](#64-nettoyage-de-clé-usb)
   - 6.5 [Gestion des badges RFID](#65-gestion-des-badges-rfid)
   - 6.6 [Mise à jour de la base ClamAV](#66-mise-à-jour-de-la-base-clamav)
   - 6.7 [Mode de sécurité](#67-mode-de-sécurité)
   - 6.8 [Gestion du PIN administrateur](#68-gestion-du-pin-administrateur)
   - 6.9 [Journaux (logs)](#69-journaux-logs)
   - 6.10 [Alimentation (arrêt / redémarrage)](#610-alimentation-arrêt--redémarrage)
   - 6.11 [Serveur de fichiers](#611-serveur-de-fichiers)
7. [Modèles de données (BDD)](#7-modèles-de-données-bdd)
8. [Routes URL](#8-routes-url)
9. [Scripts utilitaires](#9-scripts-utilitaires)
10. [Déploiement en production](#10-déploiement-en-production)
11. [Choses importantes à savoir](#11-choses-importantes-à-savoir)
12. [Pistes d'amélioration](#12-pistes-damélioration)

---

## 1. Présentation du projet

**Station Blanche Web** est une application web Django conçue pour sécuriser le transfert de fichiers via clé USB dans un environnement contrôlé (type poste de décontamination USB). Elle fonctionne sur une tablette ou un PC dédié (résolution cible : 1280×800).

### Objectif principal

Permettre à des utilisateurs authentifiés par badge RFID de :
- Insérer une clé USB
- La faire scanner automatiquement par l'antivirus ClamAV
- Copier les fichiers sains vers une autre clé USB ou un serveur de fichiers réseau
- Nettoyer les fichiers infectés

### Cas d'usage typique

1. L'utilisateur scanne son badge RFID → connexion automatique
2. Il insère une seule clé USB → détection automatique
3. L'application copie le contenu dans un dossier de staging sur disque (`/var/lib/station-blanche/staging/`)
4. ClamAV analyse les fichiers
5. Si des virus sont détectés : l'utilisateur choisit de supprimer ou d'ignorer
6. Les fichiers sains sont copiés vers une clé USB cible ou vers le serveur de fichiers

---

## 2. Architecture technique

```
┌────────────────────┐
│   Navigateur Web   │  ← Interface utilisateur (tablette/PC)
│   (port 80)        │
└────────┬───────────┘
         │  HTTP
┌────────▼───────────┐
│   Apache2          │  ← Reverse proxy
│   (mod_proxy)      │
└────────┬───────────┘
         │  HTTP interne
┌────────▼───────────┐
│   Gunicorn         │  ← Serveur WSGI Python
│   (port 8000)      │
└────────┬───────────┘
         │
┌────────▼───────────┐
│   Django 5.2       │  ← Application web
│   + SQLite         │
└────────┬───────────┘
         │
    ┌────┴────┐
    │         │
ClamAV    lsblk/parted   ← Outils système
```

### Technologies utilisées

| Composant         | Technologie                         |
|-------------------|-------------------------------------|
| Framework web     | Django 5.2.2                        |
| Serveur WSGI      | Gunicorn 23.0.0                     |
| Reverse proxy     | Apache2 (mod_proxy, mod_proxy_http) |
| Base de données   | SQLite3                             |
| Antivirus         | ClamAV (clamscan, base hors-ligne)  |
| Gestion USB       | lsblk, pyparted, psutil            |
| Authentification  | Badges RFID (hashés PBKDF2)         |
| OS cible          | Linux (Debian/Ubuntu)               |
| Python            | 3.12+                               |

---

## 3. Structure des fichiers

```
station-blanche-web/
├── README.md                     # Présentation rapide
├── DOCUMENTATION.md              # CE FICHIER — documentation complète
├── requirements.txt              # Dépendances Python
├── Start_App.sh                  # Script de déploiement production
├── station-blanche-web.conf      # Configuration Apache2 (VirtualHost)
├── station-blanche-web.service   # Service systemd (Gunicorn)
├── error-pages/
│   └── error.html                # Page d'erreur Apache personnalisée
│
└── sb_proj/                      # Racine du projet Django
    ├── manage.py                 # CLI Django
    ├── db.sqlite3                # Base de données SQLite
    ├── clamav_script.py          # Moteur de scan ClamAV
    ├── script_update_clamav.py   # MAJ base antivirale depuis clé USB
    ├── scripts.py                # Utilitaires (USB, copie, nettoyage)
    │
    ├── clamav_database/          # Base de signatures ClamAV (hors-ligne)
    │   ├── main.cvd
    │   ├── daily.cvd
    │   └── bytecode.cvd
    │
    ├── sb_proj/                  # Configuration Django
    │   ├── settings.py           # Paramètres (BDD, timezone, etc.)
    │   ├── urls.py               # Routes racine + handler500
    │   ├── wsgi.py               # Point d'entrée WSGI
    │   └── asgi.py               # Point d'entrée ASGI (non utilisé)
    │
    └── sb_app/                   # Application principale
        ├── models.py             # Modèles : Badge, LogEntry
        ├── views.py              # Vues (toute la logique métier)
        ├── urls.py               # Routes de l'application
        ├── admin.py              # Interface d'admin Django
        ├── apps.py               # Configuration de l'app
        │
        ├── config/
        │   └── config.json       # Configuration dynamique (mode, PIN, MAJ ClamAV)
        │
        ├── logs/                 # Fichiers CSV de scan + logs Django
        │   ├── admin.csv
        │   ├── login.csv
        │   ├── clamav.csv
        │   └── scan-YYYYMMDD-HHMMSS.csv
        │
        ├── migrations/           # Migrations Django
        │   ├── 0001_initial.py
        │   └── 0002_badge_unique_name.py
        │
        └── templates/
            ├── base.html                      # Template de base (header, nav, CSS, icônes SVG)
            ├── 500.html                       # Page erreur 500
            └── sb_app/
                ├── index.html                 # Page d'accueil (liste USB)
                ├── standard_user_login.html   # Page de scan badge
                ├── entree_pin.html            # Saisie PIN admin
                ├── config.html                # (non utilisé actuellement)
                ├── change_pin.html            # Changement PIN
                ├── error.html                 # Page d'erreur applicative
                ├── logs.html                  # Affichage logs (utilisateur)
                ├── protected.html             # Hub admin (6 tuiles)
                ├── target_copy_midstep.html   # Choix clé cible / serveur
                ├── target_copy_end.html       # Résultat copie
                ├── virus_action.html          # Action sur fichiers infectés
                ├── usb_cleanup_midstep.html   # Étape intermédiaire nettoyage
                ├── usb_cleanup_end.html       # Résultat nettoyage
                └── protected/
                    ├── protected_cards.html       # Hub gestion badges
                    ├── protected_cards_add.html   # Ajout badge (2 étapes)
                    ├── protected_cards_list.html   # Liste des badges
                    ├── protected_cards_remove.html # Suppression badge par scan
                    ├── protected_clamav.html      # MAJ ClamAV
                    ├── protected_logs.html         # Journaux admin
                    ├── protected_pin.html          # Changement PIN
                    ├── protected_power.html        # Arrêt/Redémarrage
                    └── protected_security.html     # Mode de sécurité
```

---

## 4. Installation & déploiement

### Prérequis

- **OS :** Linux (Debian/Ubuntu recommandé)
- **Python :** 3.12 ou supérieur
- **ClamAV :** Paquet `clamav` installé (`clamscan` disponible dans le PATH)
- **Apache2** (pour la production)
- **Paquets Python :** voir `requirements.txt`

### Installation en développement

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

### Déploiement en production

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

## 5. Configuration

### Fichier `sb_app/config/config.json`

Ce fichier JSON stocke la configuration dynamique de l'application :

```json
{
  "security_mode": "limited",
  "pin_hash": "pbkdf2_sha256$...",
  "clamav_last_update": "02/03/2026"
}
```

| Clé                  | Description                                                    | Valeurs possibles        |
|----------------------|----------------------------------------------------------------|--------------------------|
| `security_mode`      | Mode de copie vers clé USB cible                               | `"limited"`, `"maximum"` |
| `pin_hash`           | Hash PBKDF2 du PIN administrateur (8 chiffres)                 | Chaîne hashée            |
| `clamav_last_update` | Date de dernière mise à jour de la base ClamAV                 | `"JJ/MM/AAAA"`          |

### Fichier `sb_proj/settings.py` (extrait des paramètres importants)

| Paramètre      | Valeur                | Description                                       |
|----------------|-----------------------|---------------------------------------------------|
| `DEBUG`        | `False`               | Mode production — ne jamais mettre `True` en prod |
| `TIME_ZONE`    | `Europe/Paris`        | Fuseau horaire CET/CEST                           |
| `USE_TZ`       | `True`                | Stockage des dates en UTC, affichage en CET       |
| `DATABASES`    | SQLite3               | Base de données embarquée                         |
| `LOGIN_URL`    | `/standard_login/`    | Redirection si non authentifié                    |
| `ALLOWED_HOSTS`| `127.0.0.1, localhost`| Hôtes autorisés                                  |

### Constantes dans `views.py`

| Constante              | Valeur                              | Description                                |
|------------------------|-------------------------------------|--------------------------------------------|
| `ADMIN_USERNAME`       | `"admin"`                           | Nom de l'utilisateur Django superuser      |
| `CONFIG_PATH`          | `"sb_app/config/config.json"`       | Chemin du fichier de configuration         |
| `FILESERVER_MOUNT`     | `"/mnt/fileshare"`                  | Point de montage du serveur de fichiers    |
| `FILESERVER_HEALTH_DIR`| `"/mnt/fileshare/health_check/files"`| Dossier de test de santé fileserver       |
| `FILESERVER_IP`        | `"159.31.247.100"`                  | Adresse IP du serveur de fichiers          |

### Constantes dans `scripts.py`

| Constante              | Valeur                                      | Description                                             |
|------------------------|---------------------------------------------|---------------------------------------------------------|
| `STAGING_DIR`          | `"/var/lib/station-blanche/staging"`         | Répertoire de staging sur disque (NVMe) pour les scans  |

---

## 6. Fonctionnalités détaillées

### 6.1 Authentification & contrôle d'accès

L'application utilise **deux niveaux d'authentification** :

#### Utilisateur standard (badge RFID)
- **URL :** `/standard_login/`
- L'utilisateur scanne son badge RFID via un lecteur USB (qui agit comme un clavier)
- L'identifiant du badge est comparé (via `check_password` PBKDF2) à tous les badges enregistrés en BDD
- Si trouvé → connexion en tant qu'utilisateur Django `user` + stockage du nom du badge en session
- Accès : page d'accueil, scan USB, copie de fichiers

#### Administrateur (PIN 8 chiffres)
- **URL :** `/pin/`
- Saisie d'un PIN à 8 chiffres sur un clavier virtuel
- Le PIN est vérifié contre le hash PBKDF2 stocké dans `config.json`
- Si valide → connexion en tant qu'utilisateur Django `admin` (superuser)
- Accès : toutes les pages + hub d'administration `/protected/`

#### Décorateurs de protection
- `@login_required` — nécessite une connexion (badge ou admin)
- `@superuser_required` — nécessite un compte superuser (admin uniquement)

### 6.2 Gestion des clés USB — Scan antivirus

#### Flux principal
1. **Détection USB** (`scripts.lister_cles_usb()`) : utilise `lsblk -J` pour lister les périphériques USB montés
2. **Condition :** exactement 1 clé USB doit être insérée et la base ClamAV doit être présente
3. **Copie vers le staging** (`scripts.copier_contenu_cle()`) : copie intégrale de la clé dans `/var/lib/station-blanche/staging/<nom-montage>` (sur le disque NVMe, pas en RAM)
4. **Scan ClamAV** (`clamav_script.run_scan()`) : analyse avec `clamscan -r` en utilisant la base locale `clamav_database/`
5. **Résultat :**
   - 0 virus → redirection vers la page de copie
   - N virus → page d'action sur les fichiers infectés

#### Protection anti-NVMe
Le script vérifie que le périphérique n'est pas un disque NVMe pour empêcher toute opération accidentelle sur le disque système.

### 6.3 Copie de fichiers (USB & serveur de fichiers)

Après le scan, l'utilisateur peut copier les fichiers vers :

#### Vers une clé USB cible
- **URL :** `/copy_tmp_to_key/`
- L'utilisateur insère une ou plusieurs clés USB cibles
- Les fichiers sont copiés dans un dossier nommé `analyse_JJ_MM_AA_IDXX` (ID auto-incrémenté)
- En mode **maximum** : la clé cible est vidée avant la copie
- En mode **limited** : la copie s'ajoute au contenu existant
- `os.sync()` est appelé après la copie pour forcer l'écriture sur le disque

#### Vers le serveur de fichiers
- **URL :** `/copy_tmp_to_fileserver/`
- Les fichiers sont copiés vers `/mnt/fileshare/<nom_badge>/files/analyse_JJ_MM_AA_IDXX`
- Le dossier de l'utilisateur doit pré-exister sur le serveur (sinon erreur)
- Vérification préalable de l'accessibilité du serveur (ping + test d'écriture)

### 6.4 Nettoyage de clé USB

- **URL :** `/process_key_cleanup`
- Copie la clé dans le répertoire de staging sur disque, lance un scan, puis propose de supprimer les fichiers infectés
- Les fichiers sont supprimés à la fois dans la copie staging ET sur la clé USB source
- Utilise `scripts.delete_files()` qui parcourt récursivement le dossier

### 6.5 Gestion des badges RFID

#### Hub des badges
- **URL :** `/protected/cards`
- Affiche le nombre de badges enregistrés et 3 actions : Ajouter, Supprimer, Lister

#### Ajout d'un badge (2 étapes)
- **URL :** `/protected/cards/add`
- **Étape 1 :** Saisie du nom du badge via un clavier virtuel (AZERTY)
  - Le nom doit être **unique** (vérification insensible à la casse)
  - Si un badge avec le même nom existe déjà, un message d'erreur est affiché
- **Étape 2 :** Scan du badge physique
  - L'identifiant RFID est hashé avec PBKDF2 (`make_password()`)
  - Vérification que le badge physique n'est pas déjà enregistré

> **Sécurité :** Les identifiants des badges ne sont jamais stockés en clair. Seul le hash PBKDF2 est enregistré dans le champ `texte` du modèle `Badge`.

#### Suppression d'un badge
- **Par scan :** `/protected/cards/remove` — scanner le badge à supprimer
- **Par liste :** `/protected/cards/list` — cliquer sur le bouton Supprimer à côté du nom

#### Liste des badges
- **URL :** `/protected/cards/list`
- Affiche tous les badges avec leur nom et un bouton de suppression

### 6.6 Mise à jour de la base ClamAV

- **URL :** `/protected/clamav`
- La base ClamAV fonctionne **hors-ligne** (pas de `freshclam`)
- **Procédure de mise à jour :**
  1. Télécharger les 3 fichiers de signature sur un autre PC : `main.cvd`, `daily.cvd`, `bytecode.cvd`
  2. Les mettre sur une clé USB
  3. Insérer cette clé dans la station blanche
  4. Aller dans Paramètres → Base ClamAV → Mettre à jour
  5. Le script copie les fichiers de la clé vers `sb_proj/clamav_database/`
  6. La date de mise à jour est enregistrée dans `config.json`

> **Condition :** Exactement 1 clé USB montée, contenant les 3 fichiers `.cvd`.

### 6.7 Mode de sécurité

- **URL :** `/protected/security`
- Deux modes disponibles :

| Mode      | Comportement lors de la copie vers clé USB cible         |
|-----------|----------------------------------------------------------|
| `limited` | Copie les fichiers à côté du contenu existant            |
| `maximum` | **Efface tout le contenu** de la clé cible avant copie   |

### 6.8 Gestion du PIN administrateur

- **URL :** `/protected/pin` ou `/change_pin/`
- Le PIN doit être **exactement 8 chiffres**
- Le PIN actuel est requis pour le modifier (sauf premier paramétrage)
- Stocké en PBKDF2 dans `config.json` (pas dans la BDD Django)

### 6.9 Journaux (logs)

- **URL admin :** `/protected/logs`
- **URL utilisateur :** `/logs/`
- Les logs sont stockés dans la base de données (modèle `LogEntry`)
- Affichage des 200 ou 500 dernières entrées

#### Catégories de logs

| Catégorie | Description                     |
|-----------|---------------------------------|
| `AUTH`    | Connexion / déconnexion         |
| `BADGE`   | Ajout / suppression de badge    |
| `SCAN`    | Résultats d'analyse antivirus   |
| `COPY`    | Copie de fichiers               |
| `CONFIG`  | Changement de configuration     |
| `CLAMAV`  | Mise à jour base antivirale     |
| `POWER`   | Arrêt / redémarrage             |
| `SYSTEM`  | Divers                          |

#### Niveaux de criticité

| Niveau    | Usage                                    |
|-----------|------------------------------------------|
| `INFO`    | Opération réussie                        |
| `WARNING` | Tentative échouée ou virus détecté       |
| `ERROR`   | Erreur technique                         |

#### Fuseau horaire
Les timestamps sont stockés en UTC dans la base de données mais **affichés en heure CET (Europe/Paris)** grâce au paramètre `TIME_ZONE = 'Europe/Paris'` dans `settings.py`.

### 6.10 Alimentation (arrêt / redémarrage)

- **URL :** `/protected/power`
- Boutons d'arrêt et de redémarrage du système
- Exécute `sudo systemctl poweroff` ou `sudo systemctl reboot`
- Nécessite que l'utilisateur système ait les droits sudo sur ces commandes

### 6.11 Serveur de fichiers

La station blanche peut copier des fichiers vers un serveur de fichiers réseau monté sur `/mnt/fileshare/`.

#### Vérification de santé
La fonction `check_fileserver()` vérifie :
1. **Test d'écriture** : crée un fichier temporaire dans `/mnt/fileshare/health_check/files/`
2. **Si échec → ping** : teste si le serveur est joignable sur `159.31.247.100`
3. **Résultats :**
   - `ok` : serveur accessible et permissions OK
   - `droits` : serveur joignable mais problème de permissions
   - `indispo` : serveur injoignable

#### Structure sur le serveur
```
/mnt/fileshare/
└── <nom_badge>/
    └── files/
        ├── analyse_02_03_26_ID01/
        ├── analyse_02_03_26_ID02/
        └── ...
```

---

## 7. Modèles de données (BDD)

### Badge

| Champ   | Type          | Description                                      |
|---------|---------------|--------------------------------------------------|
| `id`    | BigAutoField  | Clé primaire auto-incrémentée                    |
| `name`  | CharField(255)| Nom du badge — **unique** (insensible à la casse)|
| `texte` | TextField     | Hash PBKDF2 de l'identifiant RFID                |

### LogEntry

| Champ      | Type              | Description                        |
|------------|-------------------|------------------------------------|
| `id`       | BigAutoField      | Clé primaire                       |
| `timestamp`| DateTimeField     | Date/heure (auto, indexée)         |
| `event`    | CharField(100)    | Nom de l'événement                 |
| `details`  | TextField         | Description détaillée              |
| `level`    | CharField(10)     | INFO / WARNING / ERROR             |
| `category` | CharField(20)     | AUTH / BADGE / SCAN / COPY / etc.  |
| `user`     | CharField(150)    | Nom de l'utilisateur concerné      |

---

## 8. Routes URL

### Routes utilisateur

| URL                        | Méthode    | Vue                         | Description                              |
|----------------------------|------------|------------------------------|------------------------------------------|
| `/`                        | GET        | `index`                      | Page d'accueil, détection USB            |
| `/standard_login/`         | GET, POST  | `standard_user_process`      | Connexion par badge RFID                 |
| `/pin/`                    | GET, POST  | `entree_pin`                 | Connexion admin par PIN                  |
| `/logout/`                 | GET        | `disconnect_user`            | Déconnexion                              |
| `/process_key_to_tmp/`     | POST       | `process_key_to_tmp`         | Lance scan + copie vers staging (disque) |
| `/copy_key_to_tmp/`        | GET        | `copy_key_to_tmp_view`       | Choix de la destination (clé/serveur)    |
| `/copy_tmp_to_key/`        | POST       | `copy_tmp_to_key_view`       | Copie vers clé(s) USB cible(s)           |
| `/copy_tmp_to_fileserver/` | POST       | `copy_tmp_to_fileserver_view`| Copie vers serveur de fichiers           |
| `/virus_action/`           | GET, POST  | `virus_action_view`          | Action sur fichiers infectés             |
| `/process_key_cleanup`     | POST       | `process_key_cleanup`        | Lance nettoyage USB                      |
| `/usb_cleanup_midstep`     | GET        | `usb_cleanup_midstep_view`   | Étape intermédiaire nettoyage            |
| `/cancel_cleanup`          | POST       | `cancel_cleanup_view`        | Annuler le nettoyage                     |
| `/cleanup_approved`        | POST       | `cleanup_approved_view`      | Confirmer la suppression des virus       |
| `/logs/`                   | GET        | `log_view`                   | Journaux (vue utilisateur)               |
| `/shutdown`                | GET        | `shutdown`                   | Éteindre la machine                      |
| `/reboot`                  | GET        | `reboot`                     | Redémarrer la machine                    |

### Routes administrateur (nécessitent `superuser`)

| URL                        | Méthode    | Vue                         | Description                              |
|----------------------------|------------|------------------------------|------------------------------------------|
| `/protected/`              | GET        | `protected_view`             | Hub admin (6 tuiles)                     |
| `/protected/cards`         | GET        | `protected_cards`            | Hub gestion badges                       |
| `/protected/cards/add`     | GET, POST  | `protected_cards_add`        | Ajout badge (2 étapes)                   |
| `/protected/cards/remove`  | GET, POST  | `protected_cards_remove`     | Suppression badge par scan               |
| `/protected/cards/list`    | GET, POST  | `protected_cards_list`       | Liste badges + suppression               |
| `/protected/clamav`        | GET, POST  | `protected_clamav`           | Mise à jour base ClamAV                  |
| `/protected/logs`          | GET        | `protected_logs`             | Journaux système (500 entrées)           |
| `/protected/pin`           | GET, POST  | `change_pin_view`            | Changement de PIN                        |
| `/protected/power`         | GET, POST  | `protected_power`            | Arrêt / Redémarrage                      |
| `/protected/security`      | GET, POST  | `protected_security`         | Mode de sécurité                         |
| `/change_pin/`             | GET, POST  | `change_pin_view`            | Changement de PIN (ancienne route)       |

---

## 9. Scripts utilitaires

### `clamav_script.py`
- **`run_scan(folder_path, action, ...)`** : Point d'entrée principal pour le scan
  - `action` : `"scan"` (analyse seule), `"delete"` (supprime les infectés), `"quarantine"` (déplace)
  - Génère des logs CSV dans `sb_app/logs/`
  - Retourne `(code_retour, liste_fichiers_infectés)`

### `script_update_clamav.py`
- **`update()`** : Copie les fichiers `.cvd` depuis une clé USB vers `clamav_database/`
  - Vérifie qu'exactement 1 clé est montée
  - Vérifie la présence des 3 fichiers requis
  - Retourne une chaîne `"Succès: ..."` ou `"Échec: ..."`

### `scripts.py`
- **`lister_cles_usb()`** : Liste les clés USB avec taille, point de montage, etc.
- **`copier_contenu_cle(device, mount)`** : Copie le contenu d'une clé vers le répertoire de staging sur disque (`/var/lib/station-blanche/staging/`)
- **`copier_dossier_sur_usb(source, mount, mode)`** : Copie vers une clé USB cible
- **`copier_dossier_sur_fileserver(source, badge_name)`** : Copie vers le serveur de fichiers
- **`clear_mount_point(mount)`** : Vide une clé USB (mode maximum) avec protection anti-NVMe
- **`delete_files(fichiers, dossier)`** : Supprime récursivement des fichiers par nom
- **`is_nvme_device(mount)`** : Vérifie si un point de montage est sur un NVMe

---

## 10. Déploiement en production

### Service systemd (`station-blanche-web.service`)
- Exécute Gunicorn comme service système
- Chemin du venv : `/home/kiosk/station-blanche-web/.venv`
- Répertoire de travail : `/home/kiosk/station-blanche-web/sb_proj`
- Port : 8000 (interne uniquement)

### Configuration Apache (`station-blanche-web.conf`)
- Reverse proxy HTTP sur le port 80 → Gunicorn sur le port 8000
- Pages d'erreur personnalisées (`error-pages/error.html`)

### Permissions requises
- L'utilisateur système (ex: `kiosk`) doit pouvoir exécuter :
  - `sudo systemctl poweroff`
  - `sudo systemctl reboot`
  - Lecture/écriture sur `/mnt/fileshare/`
  - Lecture des périphériques USB (`lsblk`, montage)  - Lecture/écriture sur `/var/lib/station-blanche/staging/` (répertoire de staging pour les scans)- La BDD SQLite doit être accessible en écriture : `chmod 664 db.sqlite3`
- Le dossier de logs : `chmod -R 775 sb_app/logs/`

---

## 11. Choses importantes à savoir

### Sécurité des badges
- Les identifiants RFID ne sont **jamais stockés en clair** — uniquement en hash PBKDF2
- La comparaison se fait en bouclant sur tous les badges et en testant chaque hash (ce qui fonctionne mais est O(n) — acceptable pour un petit nombre de badges)
- **Les noms de badges doivent être uniques** (contrainte en BDD + vérification côté vue, insensible à la casse)

### Base ClamAV hors-ligne
- La station blanche n'a **pas accès à Internet** → pas de `freshclam`
- Les fichiers `.cvd` doivent être récupérés manuellement et copiés via clé USB
- L'ancienneté de la base est affichée sur la page d'accueil (nombre de jours depuis la dernière MAJ)

### Gestion de la session
- Données stockées en session : `key_name`, `key_mount`, `folder`, `badge_user_name`, `scan_result`, `scan_virus`, `malicious_files_list`, `badge_add_step`, `badge_add_name`
- La session est vidée à la déconnexion (`request.session.flush()`)

### Copie temporaire (staging)
- Les fichiers sont d'abord copiés dans `/var/lib/station-blanche/staging/` (sur le disque NVMe) avant le scan
- **Pourquoi pas `/tmp` ?** Sur ce système, `/tmp` est un tmpfs (système de fichiers en RAM). Copier une clé USB de 16+ Go dans `/tmp` saturerait les 8 Go de RAM et provoquerait un crash OOM (Out Of Memory)
- Le répertoire de staging est sur le disque NVMe (441 Go disponibles), ce qui permet de traiter des clés USB de grande capacité sans impacter la RAM
- Le dossier de staging est nettoyé (`shutil.rmtree`) après la copie vers la destination finale
- **Attention :** si l'utilisateur quitte le flux en cours de route, le dossier de staging peut rester. Pas de nettoyage automatique en cas d'abandon.

### Utilisateurs Django
- `admin` (superuser) : créé via `createsuperuser`, utilisé pour l'authentification PIN
- `user` (utilisateur standard) : créé manuellement, utilisé pour l'authentification badge
- **Ne pas supprimer ces utilisateurs** sinon l'authentification ne fonctionnera plus

### Fuseau horaire
- Le fuseau est **Europe/Paris** (CET en hiver, CEST en été)
- Django stocke les dates en UTC en BDD (`USE_TZ = True`) mais les affiche dans le fuseau configuré
- Les fichiers CSV de scan utilisent l'heure locale (pas UTC)

### Secret key Django
- Générée automatiquement et stockée dans `sb_proj/.secret_key`
- **Ne pas supprimer ce fichier** en production (invaliderait toutes les sessions)

### Debug
- `DEBUG = False` en production — les erreurs 500 sont gérées par un handler personnalisé
- Les erreurs Django sont loguées dans `sb_app/logs/django-errors.log`

---

## 12. Pistes d'amélioration

Voici des axes de travail suggérés pour les futures équipes :

1. **Nettoyage automatique du staging** : Implémenter un mécanisme (middleware ou tâche périodique) pour supprimer les dossiers orphelins dans `/var/lib/station-blanche/staging/`

2. **Pagination des logs** : Actuellement limités à 200/500 entrées — ajouter une pagination côté serveur et des filtres (par date, catégorie, niveau)

3. **Export des logs** : Permettre l'export CSV/PDF des journaux depuis l'interface admin

4. **Timeout de session** : Configurer un timeout de session automatique pour éviter qu'un utilisateur reste connecté indéfiniment

5. **Multi-badges** : Optimiser la vérification des badges (actuellement en O(n) sur tous les badges) — potentiellement avec un index ou une table de lookup

6. **Tests automatisés** : Écrire des tests unitaires et d'intégration (`sb_app/tests.py` est vide)

7. **Notifications** : Alerter l'administrateur quand la base ClamAV est trop ancienne (>30 jours par exemple)

8. **Internationalisation** : Le code est en français, ajouter le support multi-langue via Django i18n si nécessaire

9. **HTTPS** : Configurer Apache avec un certificat SSL/TLS si la station est accessible sur un réseau

10. **Mise à jour ClamAV en ligne** : Si la station a accès à Internet dans le futur, intégrer `freshclam`

11. **Gestion des erreurs USB** : Améliorer la gestion des cas limites (clé USB retirée pendant la copie, etc.)

12. **Clavier virtuel amélioré** : Ajouter les chiffres, caractères spéciaux et majuscules au clavier virtuel d'ajout de badge

---

## Annexe : Commandes utiles

```bash
# Vérifier le statut des services
sudo systemctl status station-blanche-web
sudo systemctl status apache2

# Voir les logs Gunicorn
sudo journalctl -u station-blanche-web -f

# Voir les logs Apache
sudo tail -f /var/log/apache2/station-blanche-*.log

# Appliquer les migrations manuellement
cd /home/kiosk/station-blanche-web/sb_proj
source ../.venv/bin/activate
python3 manage.py migrate

# Accéder au shell Django
python3 manage.py shell

# Lister les badges en BDD
python3 manage.py shell -c "from sb_app.models import Badge; print(list(Badge.objects.values_list('name', flat=True)))"

# Vérifier les fichiers ClamAV
ls -la clamav_database/

# Redémarrer les services après modification du code
sudo systemctl restart station-blanche-web
sudo systemctl restart apache2
```
