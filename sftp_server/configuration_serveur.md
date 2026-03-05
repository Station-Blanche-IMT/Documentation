---
title: Configuration du serveur SFTP
layout: default
parent: Serveur SFTP
nav_order: 1
---

# Configuration du serveur SFTP
{: .no_toc }

**Cible :** Rocky Linux / RHEL
**Objectif :** Mettre en place un serveur SFTP chrooté pour le partage de fichiers avec la Station Blanche.

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prérequis

- Rocky Linux ou RHEL installé et à jour
- Accès root ou sudo

---

## Étape 1 : Mise à jour du système et paquets utiles

```bash
sudo dnf update && sudo dnf upgrade -y
sudo dnf install dnf-automatic vim -y
```

---

## Étape 2 : Création de l'arborescence SFTP

```bash
sudo mkdir /sftp
sudo chmod 755 /sftp
sudo mkdir -p /sftp/share
sudo chown root:root /sftp
sudo chown root:root /sftp/share
sudo chmod 755 /sftp/share
```

> Le dossier racine du chroot (`/sftp` et `/sftp/share`) **doit appartenir à root** pour que le chroot SSH fonctionne.
{: .important }

---

## Étape 3 : Création des groupes et de l'utilisateur SFTP

```bash
sudo groupadd sftp
sudo groupadd sftpmaster
sudo useradd -s /bin/bash -d /home/sftp -m -g sftpmaster sftp
sudo passwd sftp
```

> L'utilisateur `sftp` est créé avec un shell bash pour la phase de configuration initiale. Il sera verrouillé à l'étape suivante.
{: .note }

---

## Étape 4 : Configuration de la Station puis verrouillage du compte

Après avoir configuré la Station Blanche (voir [Configuration côté Station](configuration_station)), verrouiller le shell de l'utilisateur :

```bash
sudo usermod -s /sbin/nologin sftp
```

---

## Étape 5 : Configuration SSH / SFTP

Éditer le fichier de configuration SSH :

```bash
sudo vim /etc/ssh/sshd_config
```

Ajouter/modifier les directives suivantes :

```
Port 6132
DebianBanner no
Subsystem sftp internal-sftp

Match Group sftp
    X11Forwarding no
    AllowTcpForwarding no
    PermitTTY no
    PermitTunnel no
    ForceCommand internal-sftp -R
    ChrootDirectory %h

Match user sftp
    ChrootDirectory /sftp/share
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

Redémarrer le service :

```bash
sudo systemctl restart sshd
```

---

## Étape 6 : Mises à jour automatiques de sécurité

### Override du timer dnf-automatic

```bash
sudo mkdir -p /etc/systemd/system/dnf-automatic.timer.d
sudo vim /etc/systemd/system/dnf-automatic.timer.d/override.conf
```

Contenu :

```ini
[Unit]
Description=dnf-automatic timer
ConditionPathExists=!/run/ostree-booted
Wants=network-online.target

[Timer]
OnCalendar=*-*-* 3:00
RandomizedDelaySec=10m
Persistent=true

[Install]
WantedBy=timers.target
```

### Configuration de dnf-automatic

```bash
sudo vim /etc/dnf/automatic.conf
```

Paramètres à modifier :

```ini
upgrade_type = security
download_updates = yes
apply_updates = yes
reboot = when-needed
```

### Activation

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now dnf-automatic.timer
sudo systemctl restart dnf-automatic
```

---

## Ajout d'utilisateurs SFTP

Utiliser le script `add_sftp_user.sh` pour ajouter automatiquement de nouveaux utilisateurs autorisés à se connecter au serveur.
