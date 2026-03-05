---
title: Accueil
layout: home
nav_order: 1
---

# Documentation — Station Blanche

Bienvenue dans le répertoire de documentation du projet **Station Blanche**. Vous trouverez ici toutes les instructions nécessaires à l'installation et la configuration des différentes composantes du système.

---

## Sections

### Système
{: .text-gamma }

Guides d'installation et de configuration du Raspberry Pi et de ses services.

| Document | Description |
|:---------|:------------|
| [Installation de la Station Blanche](systeme/installation_et_configuration_station_blanche) | Installation de Raspberry Pi OS sur NVMe |
| [Mode Kiosk](systeme/etablissement_du_mode_kiosk) | Démarrage automatique en plein écran |
| [Hotspot Wi-Fi](systeme/configuration_du_hotspot_wifi) | Configuration du point d'accès Wi-Fi |
| [ClamAV Anti-Virus](systeme/installation_de_clamav_anti_virus) | Installation et configuration de ClamAV |
| [Montage automatique USB](systeme/montage_auto_cle_USB) | Montage automatique des clés USB via udev |

### Application Web
{: .text-gamma }

Documentation complète de l'application Django.

| Document | Description |
|:---------|:------------|
| [Présentation & Architecture](web/presentation) | Vue d'ensemble du projet et stack technique |
| [Structure des fichiers](web/structure_fichiers) | Arborescence complète du projet Django |
| [Installation & Déploiement](web/installation) | Setup dev / prod et script de déploiement |
| [Configuration](web/configuration) | Fichiers de config, constantes et paramètres |
| [Fonctionnalités](web/fonctionnalites) | Authentification, scan USB, copie, badges, logs… |
| [Référence technique](web/reference_technique) | Modèles BDD, routes URL et scripts utilitaires |
| [Notes importantes](web/notes_importantes) | Points critiques pour la maintenance |
| [Pistes d'amélioration](web/ameliorations) | Axes de travail futurs et commandes utiles |

### Serveur SFTP
{: .text-gamma }

Installation et configuration du serveur de fichiers SFTP.

| Document | Description |
|:---------|:------------|
| [Configuration du serveur SFTP](sftp_server/configuration_serveur) | Mise en place du serveur SFTP chrooté (Rocky/RHEL) |
| [Configuration côté Station](sftp_server/configuration_station) | Montage SSHFS sur la Station Blanche |
