---
title: Référence technique
layout: default
parent: Application Web
nav_order: 6
---

# Référence technique
{: .no_toc }

Modèles de données, routes URL et scripts utilitaires.

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Modèles de données (BDD)

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

## Routes URL

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

## Scripts utilitaires

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
