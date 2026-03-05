---
title: Notes importantes
layout: default
parent: Application Web
nav_order: 7
---

# Choses importantes à savoir
{: .no_toc }

Points critiques pour la maintenance et l'exploitation de la Station Blanche Web.

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Sécurité des badges

- Les identifiants RFID ne sont **jamais stockés en clair** — uniquement en hash PBKDF2
- La comparaison se fait en bouclant sur tous les badges et en testant chaque hash (ce qui fonctionne mais est O(n) — acceptable pour un petit nombre de badges)
- **Les noms de badges doivent être uniques** (contrainte en BDD + vérification côté vue, insensible à la casse)

---

## Base ClamAV hors-ligne

- La station blanche n'a **pas accès à Internet** → pas de `freshclam`
- Les fichiers `.cvd` doivent être récupérés manuellement et copiés via clé USB
- L'ancienneté de la base est affichée sur la page d'accueil (nombre de jours depuis la dernière MAJ)

---

## Gestion de la session

- Données stockées en session : `key_name`, `key_mount`, `folder`, `badge_user_name`, `scan_result`, `scan_virus`, `malicious_files_list`, `badge_add_step`, `badge_add_name`, `copy_task_id`
- `copy_task_id` : identifiant UUID de la tâche de copie en cours (utilisé par la page de progression)
- La session est vidée à la déconnexion (`request.session.flush()`)

---

## Copie temporaire (staging)

- Les fichiers sont d'abord copiés dans `/var/lib/station-blanche/staging/` (sur le disque NVMe) avant le scan
- **Pourquoi pas `/tmp` ?** Sur ce système, `/tmp` est un tmpfs (système de fichiers en RAM). Copier une clé USB de 16+ Go dans `/tmp` saturerait les 8 Go de RAM et provoquerait un crash OOM (Out Of Memory)
- Le répertoire de staging est sur le disque NVMe (441 Go disponibles), ce qui permet de traiter des clés USB de grande capacité sans impacter la RAM
- Le dossier de staging est nettoyé (`shutil.rmtree`) après la copie vers la destination finale
- **Permissions :** le dossier est créé avec les permissions `755` pour que le démon ClamAV (qui tourne sous l'utilisateur `clamav`) puisse lire les fichiers à scanner. Des permissions plus restrictives (ex : `750`) empêchent `clamdscan` d'accéder aux fichiers.

> **Attention :** si l'utilisateur quitte le flux en cours de route, le dossier de staging peut rester. Pas de nettoyage automatique en cas d'abandon.
{: .warning }

---

## Utilisateurs Django

- `admin` (superuser) : créé via `createsuperuser`, utilisé pour l'authentification PIN
- `user` (utilisateur standard) : créé manuellement, utilisé pour l'authentification badge

> **Ne pas supprimer ces utilisateurs** sinon l'authentification ne fonctionnera plus.
{: .important }

---

## Fuseau horaire

- Le fuseau est **Europe/Paris** (CET en hiver, CEST en été)
- Django stocke les dates en UTC en BDD (`USE_TZ = True`) mais les affiche dans le fuseau configuré
- Les fichiers CSV de scan utilisent l'heure locale (pas UTC)

---

## Secret key Django

- Générée automatiquement et stockée dans `sb_proj/.secret_key`
- **Ne pas supprimer ce fichier** en production (invaliderait toutes les sessions)

---

## Debug

- `DEBUG = False` en production — les erreurs 500 sont gérées par un handler personnalisé
- Les erreurs Django sont loguées dans `sb_app/logs/django-errors.log`

---

## Gunicorn : worker unique obligatoire

- Gunicorn doit être configuré avec **`--workers 1 --threads 4`**
- Le suivi de progression des copies utilise un dictionnaire Python en mémoire (`_copy_tasks`), partagé entre les threads du même processus
- Si Gunicorn est lancé avec plusieurs workers (ex : `--workers 2`), chaque worker est un **processus séparé** avec sa propre mémoire → la requête de copie peut atterrir sur un worker et la requête de polling sur un autre, rendant la barre de progression inopérante
- Les threads, eux, partagent la mémoire du processus et permettent à la fois la copie en arrière-plan et le polling simultané

---

## Synchronisation USB et dirty pages

- Lors d'une copie vers clé USB, Linux bufferise les écritures en RAM (« dirty pages ») avant de les écrire physiquement
- Pour les fichiers volumineux, la copie applicative (`shutil.copytree`) peut terminer en quelques secondes alors que l'écriture réelle sur la clé prend plusieurs minutes
- L'application gère cela en deux phases :
  1. **Phase copie** : copie avec suivi des octets (blocs de 1 Mo), `skip_sync=True` pour ne pas appeler `os.sync()` immédiatement
  2. **Phase synchronisation** : appel `sync` + surveillance du champ `Dirty:` de `/proc/meminfo` toutes les 0,5 s jusqu'à ce que les dirty pages passent sous 1 Mo
- Cela permet à la barre de progression de refléter l'écriture réelle sur le support USB
