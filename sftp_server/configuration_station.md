---
title: Configuration côté Station
layout: default
parent: Serveur SFTP
nav_order: 2
---

# Configuration SFTP côté Station Blanche
{: .no_toc }

**Cible :** Station Blanche (Raspberry Pi / Debian)
**Objectif :** Monter le partage SFTP distant en tant que système de fichiers local via SSHFS.

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prérequis

- Station Blanche installée et à jour
- Serveur SFTP configuré (voir [Configuration du serveur SFTP](configuration_serveur))
- Connectivité réseau entre la station et le serveur

> Remplacer les champs marqués `<INFO>` par les informations correspondantes à votre environnement.
{: .warning }

---

## Étape 1 : Installation de SSHFS

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install sshfs -y
sudo apt autoremove -y
```

---

## Étape 2 : Création de l'utilisateur local

```bash
sudo useradd -s /bin/bash -d /home/user -m -G user user
sudo passwd user
```

---

## Étape 3 : Génération et copie des clés SSH

```bash
su - user
ssh-keygen -t ed25519
ssh-copy-id sftp@<IP_SERVER>
ssh-copy-id user@<IP_SERVER>
mkdir fileshare
exit
```

---

## Étape 4 : Configuration du point de montage

### Permissions

```bash
sudo chmod 700 /mnt/fileshare
```

### Autoriser le montage FUSE pour d'autres utilisateurs

```bash
sudo vim /etc/fuse.conf
```

Décommenter ou ajouter :

```
user_allow_other
```

---

## Étape 5 : Montage automatique via fstab

```bash
sudo vim /etc/fstab
```

Ajouter la ligne suivante :

```
sftp@<IP_SERVER>:/ /mnt/fileshare fuse.sshfs defaults,delay_connect,_netdev,IdentityFile=/home/user/.ssh/id_ed25519,port=6132,uid=<LOCAL_UID>,gid=<LOCAL_GID>,allow_other,reconnect,ConnectTimeout=5 0 0
```

| Paramètre | Description |
|:----------|:------------|
| `<IP_SERVER>` | Adresse IP du serveur SFTP |
| `<LOCAL_UID>` | UID de l'utilisateur local (obtenir avec `id -u user`) |
| `<LOCAL_GID>` | GID du groupe local (obtenir avec `id -g user`) |
| `port=6132` | Port SSH configuré sur le serveur |
| `allow_other` | Permet aux autres utilisateurs d'accéder au montage |
| `reconnect` | Reconnexion automatique en cas de perte réseau |

---

## Étape 6 : Redémarrage

```bash
sudo reboot
```

Après le redémarrage, le partage SFTP sera automatiquement monté sur `/mnt/fileshare`.
