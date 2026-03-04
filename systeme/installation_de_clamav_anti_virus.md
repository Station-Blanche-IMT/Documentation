---
title: ClamAV Anti-Virus
layout: default
parent: Système
nav_order: 4
---

# Installation et Configuration de ClamAV

**Cible :** Raspberry Pi 5 / Raspberry Pi OS Bookworm
**Objectif :** Installer ClamAV en mode démon (`clamdscan`) pour l'analyse antivirale des fichiers uploadés par l'application Django.

---

## Prérequis

- Raspberry Pi OS Bookworm installé.
- Accès Internet pour le téléchargement des paquets et des bases virales.
- L'application Django déployée dans `/home/kiosk/station-blanche-web/`.

---

## Étape 1 : Installer ClamAV

```bash
sudo apt install clamav clamav-daemon -y
```

| Paquet | Rôle |
|---|---|
| `clamav` | Moteur antivirus + outil `clamscan` (scan en ligne de commande) |
| `clamav-daemon` | Démon `clamd` + outil `clamdscan` (scan via socket, beaucoup plus rapide) |

> **Pourquoi `clamav-daemon` ?**
> L'application utilise `clamdscan` (et non `clamscan`). `clamdscan` communique avec le démon `clamd` qui garde les bases virales en mémoire. Chaque scan est instantané car il n'y a pas de rechargement des bases. `clamscan` recharge les bases à chaque appel (plusieurs secondes de délai).

---

## Étape 2 : Mettre à jour les bases virales (via Internet)

### Méthode online (recommandée dès qu'une connexion est disponible)

```bash
sudo systemctl stop clamav-freshclam
sudo freshclam
sudo systemctl start clamav-freshclam
```

> `freshclam` est le service de mise à jour automatique. On l'arrête avant la mise à jour manuelle pour éviter les conflits de verrou sur les fichiers de base.

**Vérification :**

```bash
sudo systemctl status clamav-freshclam
```

Les bases sont stockées dans `/var/lib/clamav/` (`main.cvd`, `daily.cvd`, `bytecode.cvd`).

---

## Étape 3 : Installer les bases virales locales (pour l'application)

L'application Django utilise un dossier `clamav_database/` local situé dans `sb_proj/` pour ses scans. Ce dossier doit contenir les trois fichiers de base virale.

### Aller dans le répertoire de l'application

```bash
cd /home/kiosk/station-blanche-web/sb_proj
mkdir -p clamav_database
cd clamav_database
```

### Télécharger les trois bases

```bash
# 1. La base principale (grosse, rarement mise à jour)
curl -L -O https://database.clamav.net/main.cvd

# 2. La mise à jour quotidienne (change plusieurs fois par jour)
curl -L -O https://database.clamav.net/daily.cvd

# 3. Les signatures bytecode (scripts malveillants)
curl -L -O https://database.clamav.net/bytecode.cvd
```

| Fichier | Taille approximative | Fréquence de mise à jour |
|---|---|---|
| `main.cvd` | ~160 Mo | Mensuelle |
| `daily.cvd` | ~60 Mo | Plusieurs fois par jour |
| `bytecode.cvd` | ~400 Ko | Hebdomadaire |

> **Ces bases sont indépendantes** des bases système dans `/var/lib/clamav/`. Elles sont utilisées exclusivement par l'application Django via le script `clamav_script.py`.

---

## Étape 4 : Démarrer et activer le démon ClamAV

```bash
sudo systemctl enable clamav-daemon
sudo systemctl start clamav-daemon
```

**Vérification :**

```bash
sudo systemctl status clamav-daemon
```

Le démon est prêt quand le statut affiche `active (running)` et la ligne :
```
Listening daemon: PID: XXXX, Socket: /run/clamav/clamd.ctl
```

---

## Étape 5 : Tester le scanner

### Test rapide en ligne de commande (`clamscan`)

```bash
clamscan -r /home/kiosk
```

### Test via le démon (`clamdscan` — comme l'application)

```bash
clamdscan --recursive /home/kiosk
```

Un résultat propre ressemble à :

```
----------- SCAN SUMMARY -----------
Known viruses: 8631285
Engine version: 1.4.x
Scanned directories: XX
Scanned files: XX
Infected files: 0
Time: X.XXX sec (X m X s)
```

---

## Récapitulatif des fichiers importants

| Chemin | Rôle |
|---|---|
| `/var/lib/clamav/` | Bases virales système (utilisées par `clamscan` et `clamd`) |
| `/home/kiosk/station-blanche-web/sb_proj/clamav_database/` | Bases virales locales de l'application Django |
| `/home/kiosk/station-blanche-web/sb_proj/clamav_script.py` | Script Python qui appelle `clamdscan` |
| `/run/clamav/clamd.ctl` | Socket Unix du démon `clamd` |

---

## Différence clamscan vs clamdscan

| | `clamscan` | `clamdscan` |
|---|---|---|
| Fonctionnement | Charge les bases à chaque scan | Se connecte au démon `clamd` déjà en mémoire |
| Vitesse | Lent (plusieurs secondes de démarrage) | Rapide (scan quasi-instantané) |
| Prérequis | Aucun | `clamav-daemon` installé et actif |
| Usage | Tests ponctuels en CLI | **Utilisé par l'application** |

---

## Dépannage

### `clamdscan` retourne "Could not connect to clamd"
Le démon n'est pas démarré :
```bash
sudo systemctl start clamav-daemon
sudo systemctl status clamav-daemon
```

### `clamdscan` retourne "Permission denied" sur les fichiers
L'utilisateur `kiosk` doit appartenir au groupe `clamav` :
```bash
sudo usermod -aG clamav kiosk
# Se déconnecter/reconnecter ou :
newgrp clamav
```

### Les bases locales sont absentes ou corrompues
Retélécharger les fichiers `.cvd` (voir Étape 3) :
```bash
cd /home/kiosk/station-blanche-web/sb_proj/clamav_database
curl -L -O https://database.clamav.net/main.cvd
curl -L -O https://database.clamav.net/daily.cvd
curl -L -O https://database.clamav.net/bytecode.cvd
```

### `freshclam` échoue avec "locked by another process"
```bash
sudo systemctl stop clamav-freshclam
sudo rm -f /var/lock/clamav-freshclam.lock
sudo freshclam
sudo systemctl start clamav-freshclam
```
