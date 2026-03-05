---
title: Fonctionnalités
layout: default
parent: Application Web
nav_order: 5
---

# Fonctionnalités détaillées
{: .no_toc }

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Authentification & contrôle d'accès

L'application utilise **deux niveaux d'authentification** :

### Utilisateur standard (badge RFID)
- **URL :** `/standard_login/`
- L'utilisateur scanne son badge RFID via un lecteur USB (qui agit comme un clavier)
- L'identifiant du badge est comparé (via `check_password` PBKDF2) à tous les badges enregistrés en BDD
- Si trouvé → connexion en tant qu'utilisateur Django `user` + stockage du nom du badge en session
- Accès : page d'accueil, scan USB, copie de fichiers

### Administrateur (PIN 8 chiffres)
- **URL :** `/pin/`
- Saisie d'un PIN à 8 chiffres sur un clavier virtuel
- Le PIN est vérifié contre le hash PBKDF2 stocké dans `config.json`
- Si valide → connexion en tant qu'utilisateur Django `admin` (superuser)
- Accès : toutes les pages + hub d'administration `/protected/`

### Décorateurs de protection
- `@login_required` — nécessite une connexion (badge ou admin)
- `@superuser_required` — nécessite un compte superuser (admin uniquement)

---

## Gestion des clés USB — Scan antivirus

### Flux principal
1. **Détection USB** (`scripts.lister_cles_usb()`) : utilise `lsblk -J` pour lister les périphériques USB montés
2. **Condition :** exactement 1 clé USB doit être insérée et la base ClamAV doit être présente
3. **Copie vers le staging** (`scripts.copier_contenu_cle()`) : copie intégrale de la clé dans `/var/lib/station-blanche/staging/<nom-montage>` (sur le disque NVMe, pas en RAM)
4. **Scan ClamAV** (`clamav_script.run_scan()`) : analyse avec `clamscan -r` en utilisant la base locale `clamav_database/`
5. **Résultat :**
   - 0 virus → redirection vers la page de copie
   - N virus → page d'action sur les fichiers infectés

### Protection anti-NVMe
Le script vérifie que le périphérique n'est pas un disque NVMe pour empêcher toute opération accidentelle sur le disque système.

---

## Copie de fichiers (USB & serveur de fichiers)

Après le scan, l'utilisateur peut copier les fichiers vers :

### Vers une clé USB cible
- **URL :** `/copy_tmp_to_key/`
- L'utilisateur insère une ou plusieurs clés USB cibles
- Les fichiers sont copiés dans un dossier nommé `analyse_JJ_MM_AA_IDXX` (ID auto-incrémenté)
- En mode **maximum** : la clé cible est vidée avant la copie
- En mode **limited** : la copie s'ajoute au contenu existant
- `os.sync()` est appelé après la copie pour forcer l'écriture sur le disque

### Vers le serveur de fichiers
- **URL :** `/copy_tmp_to_fileserver/`
- Les fichiers sont copiés vers `/mnt/fileshare/<nom_badge>/files/analyse_JJ_MM_AA_IDXX`
- Le dossier de l'utilisateur doit pré-exister sur le serveur (sinon erreur)
- Vérification préalable de l'accessibilité du serveur (ping + test d'écriture)

---

## Nettoyage de clé USB

- **URL :** `/process_key_cleanup`
- Copie la clé dans le répertoire de staging sur disque, lance un scan, puis propose de supprimer les fichiers infectés
- Les fichiers sont supprimés à la fois dans la copie staging ET sur la clé USB source
- Utilise `scripts.delete_files()` qui parcourt récursivement le dossier

---

## Gestion des badges RFID

### Hub des badges
- **URL :** `/protected/cards`
- Affiche le nombre de badges enregistrés et 3 actions : Ajouter, Supprimer, Lister

### Ajout d'un badge (2 étapes)
- **URL :** `/protected/cards/add`
- **Étape 1 :** Saisie du nom du badge via un clavier virtuel (AZERTY)
  - Le nom doit être **unique** (vérification insensible à la casse)
  - Si un badge avec le même nom existe déjà, un message d'erreur est affiché
- **Étape 2 :** Scan du badge physique
  - L'identifiant RFID est hashé avec PBKDF2 (`make_password()`)
  - Vérification que le badge physique n'est pas déjà enregistré

> **Sécurité :** Les identifiants des badges ne sont jamais stockés en clair. Seul le hash PBKDF2 est enregistré dans le champ `texte` du modèle `Badge`.
{: .important }

### Suppression d'un badge
- **Par scan :** `/protected/cards/remove` — scanner le badge à supprimer
- **Par liste :** `/protected/cards/list` — cliquer sur le bouton Supprimer à côté du nom

### Liste des badges
- **URL :** `/protected/cards/list`
- Affiche tous les badges avec leur nom et un bouton de suppression

---

## Mise à jour de la base ClamAV

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
{: .note }

---

## Mode de sécurité

- **URL :** `/protected/security`
- Deux modes disponibles :

| Mode      | Comportement lors de la copie vers clé USB cible         |
|-----------|----------------------------------------------------------|
| `limited` | Copie les fichiers à côté du contenu existant            |
| `maximum` | **Efface tout le contenu** de la clé cible avant copie   |

---

## Gestion du PIN administrateur

- **URL :** `/protected/pin` ou `/change_pin/`
- Le PIN doit être **exactement 8 chiffres**
- Le PIN actuel est requis pour le modifier (sauf premier paramétrage)
- Stocké en PBKDF2 dans `config.json` (pas dans la BDD Django)

---

## Journaux (logs)

- **URL admin :** `/protected/logs`
- **URL utilisateur :** `/logs/`
- Les logs sont stockés dans la base de données (modèle `LogEntry`)
- Affichage des 200 ou 500 dernières entrées

### Catégories de logs

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

### Niveaux de criticité

| Niveau    | Usage                                    |
|-----------|------------------------------------------|
| `INFO`    | Opération réussie                        |
| `WARNING` | Tentative échouée ou virus détecté       |
| `ERROR`   | Erreur technique                         |

### Fuseau horaire
Les timestamps sont stockés en UTC dans la base de données mais **affichés en heure CET (Europe/Paris)** grâce au paramètre `TIME_ZONE = 'Europe/Paris'` dans `settings.py`.

---

## Alimentation (arrêt / redémarrage)

- **URL :** `/protected/power`
- Boutons d'arrêt et de redémarrage du système
- Exécute `sudo systemctl poweroff` ou `sudo systemctl reboot`
- Nécessite que l'utilisateur système ait les droits sudo sur ces commandes

---

## Serveur de fichiers

La station blanche peut copier des fichiers vers un serveur de fichiers réseau monté sur `/mnt/fileshare/`.

### Vérification de santé
La fonction `check_fileserver()` vérifie :
1. **Test d'écriture** : crée un fichier temporaire dans `/mnt/fileshare/health_check/files/`
2. **Si échec → ping** : teste si le serveur est joignable sur `159.31.247.100`
3. **Résultats :**
   - `ok` : serveur accessible et permissions OK
   - `droits` : serveur joignable mais problème de permissions
   - `indispo` : serveur injoignable

### Structure sur le serveur
```
/mnt/fileshare/
└── <nom_badge>/
    └── files/
        ├── analyse_02_03_26_ID01/
        ├── analyse_02_03_26_ID02/
        └── ...
```
