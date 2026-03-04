---
title: Installation de la Station Blanche
layout: default
parent: Système
nav_order: 1
---

# Installation de Raspberry Pi OS sur NVMe

**Cible :** Raspberry Pi 5
**Objectif :** Installer Raspberry Pi OS 64 bits sur le disque NVMe via une carte SD de démarrage et une installation réseau.

---

## Prérequis

- Une carte SD avec le firmware de démarrage Raspberry Pi (Raspberry Pi Imager suffit pour flasher le bootloader).
- Un disque NVMe installé dans le slot M.2 du Raspberry Pi 5 (via HAT ou adaptateur).
- Une connexion Internet active (câble Ethernet recommandé pour la stabilité du téléchargement).
- Écran, clavier connectés au Raspberry Pi.

# Tips :
	La carte SD n'est pas obligatoire ; il semble possible d'installer le système d'exploitation sur le NVMe via le réseau. 

---

## Étape 1 : Accéder au BIOS / Bootloader

1. Insérer la carte SD dans le Raspberry Pi.
2. Mettre le Raspberry Pi sous tension.
3. **Au démarrage**, appuyer sur **Maj (Shift)** pour accéder à l'interface du bootloader EEPROM.

> Si l'écran reste noir, appuyer sur Maj dès que l'alimentation est connectée, avant l'affichage du logo.

---

## Étape 2 : Lancer l'installation réseau

Dans l'interface du bootloader :

1. Appuyer sur **Espace** pour ouvrir le menu.
2. Appuyer sur **N** pour sélectionner l'installation via le réseau (**Network Install**).

Le Raspberry Pi va télécharger l'outil d'installation depuis Internet. Une connexion Ethernet est nécessaire à cette étape.

---

## Étape 3 : Choisir la version de l'OS

Une fois l'outil d'installation chargé :

1. Sélectionner **Raspberry Pi OS (64-bit)**.

> Choisir impérativement la version **64 bits** pour tirer parti des performances du Raspberry Pi 5 et assurer la compatibilité avec les paquets de l'application.

---

## Étape 4 : Choisir la cible d'installation

1. Sélectionner le **NVMe** comme disque de destination.

> Ne pas sélectionner la carte SD, elle sert uniquement à amorcer l'installateur.

---

## Étape 5 : Installation

1. Confirmer et lancer l'installation.
2. **Attendre** le téléchargement et l'écriture de l'image (peut prendre plusieurs minutes selon la connexion).
3. Le système redémarre automatiquement à la fin.

> Retirer la carte SD lorsque demandé, ou après le premier redémarrage, pour que le Pi démarre bien depuis le NVMe.

---

## Étape 6 : Configuration initiale de l'OS

Au premier démarrage, l'assistant de configuration (OOBE) se lance. Renseigner les informations demandées :

| Paramètre | Recommandation |
|---|---|
| Langue / Région | Selon le déploiement |
| Nom d'utilisateur | `pi` (par défaut) ou au choix |
| Mot de passe | Choisir un mot de passe fort |
| Wi-Fi | Configurer si pas d'Ethernet, sinon ignorer |
| Mise à jour automatique | Accepter |

> Les propriétés des utilisateurs (création de `kiosk`, droits sudo, etc.) seront configurées dans une étape ultérieure. Ne pas créer l'utilisateur `kiosk` ici.

---

## Étape 7 : Vérifier le démarrage sur NVMe

Une fois installé et redémarré, vérifier que le système tourne bien depuis le NVMe :

```bash
lsblk
```

Le système de fichiers racine `/` doit être monté sur une partition de type `nvme` (ex: `nvme0n1p2`), et non sur `mmcblk` (carte SD).

```bash
findmnt /
```

Exemple de résultat attendu :

```
TARGET SOURCE        FSTYPE OPTIONS
/      /dev/nvme0n1p2 ext4   rw,relatime
```

---

## Suite recommandée

Une fois l'OS installé et opérationnel :

| Étape suivante | Documentation |
|---|---|
| Configurer le mode Kiosk | `KIOSK_MODE_INSTALL.md` |
| Configurer le hotspot Wi-Fi | `HOTSPOT_WIFI_INSTALL.md` |
| Installer ClamAV | `CLAMAV_INSTALL.md` |
