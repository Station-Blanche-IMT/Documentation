---
title: Configuration
layout: default
parent: Application Web
nav_order: 4
---

# Configuration
{: .no_toc }

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Fichier `sb_app/config/config.json`

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

---

## Fichier `sb_proj/settings.py`

Extrait des paramètres importants :

| Paramètre      | Valeur                | Description                                       |
|----------------|-----------------------|---------------------------------------------------|
| `DEBUG`        | `False`               | Mode production — ne jamais mettre `True` en prod |
| `TIME_ZONE`    | `Europe/Paris`        | Fuseau horaire CET/CEST                           |
| `USE_TZ`       | `True`                | Stockage des dates en UTC, affichage en CET       |
| `DATABASES`    | SQLite3               | Base de données embarquée                         |
| `LOGIN_URL`    | `/standard_login/`    | Redirection si non authentifié                    |
| `ALLOWED_HOSTS`| `127.0.0.1, localhost`| Hôtes autorisés                                  |

---

## Constantes dans `views.py`

| Constante              | Valeur                              | Description                                |
|------------------------|-------------------------------------|--------------------------------------------|
| `ADMIN_USERNAME`       | `"admin"`                           | Nom de l'utilisateur Django superuser      |
| `CONFIG_PATH`          | `"sb_app/config/config.json"`       | Chemin du fichier de configuration         |
| `FILESERVER_MOUNT`     | `"/mnt/fileshare"`                  | Point de montage du serveur de fichiers    |
| `FILESERVER_HEALTH_DIR`| `"/mnt/fileshare/health_check/files"`| Dossier de test de santé fileserver       |
| `FILESERVER_IP`        | `"159.31.247.100"`                  | Adresse IP du serveur de fichiers          |

---

## Constantes dans `scripts.py`

| Constante              | Valeur                                      | Description                                             |
|------------------------|---------------------------------------------|---------------------------------------------------------|
| `STAGING_DIR`          | `"/var/lib/station-blanche/staging"`         | Répertoire de staging sur disque (NVMe) pour les scans  |
