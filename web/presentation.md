---
title: Présentation & Architecture
layout: default
parent: Application Web
nav_order: 1
---

# Présentation du projet
{: .no_toc }

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Vue d'ensemble

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

## Architecture technique

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
| Antivirus         | ClamAV (clamdscan, base hors-ligne) |
| Gestion USB       | lsblk, pyparted, psutil            |
| Authentification  | Badges RFID (hashés PBKDF2)         |
| OS cible          | Linux (Debian/Ubuntu)               |
| Python            | 3.12+                               |
