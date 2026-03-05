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

- Données stockées en session : `key_name`, `key_mount`, `folder`, `badge_user_name`, `scan_result`, `scan_virus`, `malicious_files_list`, `badge_add_step`, `badge_add_name`
- La session est vidée à la déconnexion (`request.session.flush()`)

---

## Copie temporaire (staging)

- Les fichiers sont d'abord copiés dans `/var/lib/station-blanche/staging/` (sur le disque NVMe) avant le scan
- **Pourquoi pas `/tmp` ?** Sur ce système, `/tmp` est un tmpfs (système de fichiers en RAM). Copier une clé USB de 16+ Go dans `/tmp` saturerait les 8 Go de RAM et provoquerait un crash OOM (Out Of Memory)
- Le répertoire de staging est sur le disque NVMe (441 Go disponibles), ce qui permet de traiter des clés USB de grande capacité sans impacter la RAM
- Le dossier de staging est nettoyé (`shutil.rmtree`) après la copie vers la destination finale

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
