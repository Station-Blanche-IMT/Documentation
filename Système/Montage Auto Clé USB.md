Voici la traduction en français de votre documentation :

# Guide de Configuration du Montage Automatique USB

> Raspberry Pi 5 — Debian 13 (Trixie) / Raspberry Pi OS — aarch64
> Testé avec le noyau `6.12.x+rpt-rpi-2712`

Ce guide permet de configurer le montage automatique d'une clé USB sur un Raspberry Pi sans interface graphique (headless) ou en mode kiosque en utilisant **udev + systemd + un script shell**. Aucun environnement de bureau n'est requis.

---

## Table des matières

1. [Prérequis](https://www.google.com/search?q=%231-pr%C3%A9requis)
2. [Créer le script de montage](https://www.google.com/search?q=%232-cr%C3%A9er-le-script-de-montage)
3. [Créer le service modèle systemd](https://www.google.com/search?q=%233-cr%C3%A9er-le-service-mod%C3%A8le-systemd)
4. [Créer les règles udev](https://www.google.com/search?q=%234-cr%C3%A9er-les-r%C3%A8gles-udev)
5. [Tout activer](https://www.google.com/search?q=%235-tout-activer)
6. [Tester](https://www.google.com/search?q=%236-tester)
7. [Dépannage](https://www.google.com/search?q=%237-d%C3%A9pannage)
8. [Comment ça marche](https://www.google.com/search?q=%238-comment-%C3%A7a-marche)
9. [Notes de sécurité](https://www.google.com/search?q=%239-notes-de-s%C3%A9curit%C3%A9)
10. [Désinstallation](https://www.google.com/search?q=%2310-d%C3%A9sinstallation)

---

## 1. Prérequis

### Paquets système

```bash
sudo apt update
sudo apt install -y exfat-fuse exfatprogs ntfs-3g util-linux

```

* **exfat-fuse / exfatprogs** — prise en charge du système de fichiers exFAT (courant sur les disques USB de grande capacité).
* **ntfs-3g** — prise en charge en lecture/écriture du NTFS (disques formatés pour Windows).
* **util-linux** — fournit `blkid`, `mount`, `umount` (généralement préinstallé).

### Utilisateur dédié (optionnel)

La configuration ci-dessous utilise `uid=1001` / `gid=1001` (utilisateur `kiosk`). Ajustez si votre utilisateur est différent :

```bash
# Vérifiez l'UID/GID de votre utilisateur
id kiosk
# uid=1001(kiosk) gid=1001(kiosk) ...

```

Remplacez `1001` par l'UID/GID de votre utilisateur dans le script de montage si nécessaire.

---

## 2. Créer le script de montage

```bash
sudo tee /usr/local/bin/usb-mount.sh > /dev/null << 'EOF'
#!/bin/bash

ACTION=$1
DEVBASE=$2
DEVICE="/dev/${DEVBASE}"
# Point de montage unique par partition (ex: /media/usb_sdb1)
MOUNT_POINT="/media/usb_${DEVBASE}"

do_mount() {
    if [ ! -b "${DEVICE}" ]; then
        echo "Périphérique ${DEVICE} introuvable" | systemd-cat -t usb-mount -p err
        return 1
    fi

    if [ ! -d "${MOUNT_POINT}" ]; then
        mkdir -p "${MOUNT_POINT}"
    fi

    # Détecter le type de système de fichiers
    FSTYPE=$(blkid -o value -s TYPE "${DEVICE}" 2>/dev/null)

    if [ -z "${FSTYPE}" ]; then
        echo "Impossible de détecter le système de fichiers sur ${DEVICE}" | systemd-cat -t usb-mount -p err
        rmdir "${MOUNT_POINT}" 2>/dev/null
        return 1
    fi

    # Options communes sécurisées : pas d'exécutables, pas de setuid, pas de nœuds de périphérique
    COMMON_OPTS="noexec,nosuid,nodev,noatime"

    case "${FSTYPE}" in
        vfat)
            mount -t vfat -o "${COMMON_OPTS},uid=1001,gid=1001,utf8,dmask=0022,fmask=0133" "${DEVICE}" "${MOUNT_POINT}"
            ;;
        exfat)
            mount -t exfat -o "${COMMON_OPTS},uid=1001,gid=1001,dmask=0022,fmask=0133" "${DEVICE}" "${MOUNT_POINT}"
            ;;
        ntfs|ntfs3)
            mount -t ntfs3 -o "${COMMON_OPTS},uid=1001,gid=1001,dmask=0022,fmask=0133" "${DEVICE}" "${MOUNT_POINT}" 2>/dev/null \
            || mount -t ntfs-3g -o "${COMMON_OPTS},uid=1001,gid=1001,dmask=0022,fmask=0133" "${DEVICE}" "${MOUNT_POINT}"
            ;;
        ext2|ext3|ext4)
            mount -t "${FSTYPE}" -o "${COMMON_OPTS}" "${DEVICE}" "${MOUNT_POINT}"
            # Système de fichiers Linux natif : correction des droits pour l'utilisateur kiosk
            chown 1001:1001 "${MOUNT_POINT}"
            ;;
        *)
            # Solution de repli : tentative de montage générique
            mount -o "${COMMON_OPTS}" "${DEVICE}" "${MOUNT_POINT}"
            ;;
    esac

    if [ $? -eq 0 ]; then
        echo "Monté ${DEVICE} (${FSTYPE}) sur ${MOUNT_POINT}" | systemd-cat -t usb-mount
    else
        echo "Échec du montage de ${DEVICE} (${FSTYPE})" | systemd-cat -t usb-mount -p err
        rmdir "${MOUNT_POINT}" 2>/dev/null
        return 1
    fi
}

do_unmount() {
    if mountpoint -q "${MOUNT_POINT}" 2>/dev/null; then
        umount -l "${MOUNT_POINT}"
        echo "Démonté ${DEVICE} de ${MOUNT_POINT}" | systemd-cat -t usb-mount
    fi
    rmdir "${MOUNT_POINT}" 2>/dev/null
}

case "${ACTION}" in
    add)
        do_mount
        ;;
    remove)
        do_unmount
        ;;
esac
EOF

```

Rendez-le exécutable :

```bash
sudo chmod +x /usr/local/bin/usb-mount.sh

```

---

## 3. Créer le service modèle systemd

Une **unité modèle** (`@.service`) est instanciée par périphérique (par exemple `usb-mount@sda1.service`).

```bash
sudo tee /etc/systemd/system/usb-mount@.service > /dev/null << 'EOF'
[Unit]
Description=Montage du disque USB sur %i
# Arrête automatiquement ce service lorsque le périphérique disparaît
BindsTo=dev-%i.device
After=dev-%i.device

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/usb-mount.sh add %i
ExecStop=/usr/local/bin/usb-mount.sh remove %i
EOF

```

Explication des paramètres clés :

| Directive | Objectif |
| --- | --- |
| `BindsTo=dev-%i.device` | Lie le cycle de vie du service au périphérique. Lorsque le périphérique disparaît, systemd arrête automatiquement le service. |
| `After=dev-%i.device` | S'assure que le périphérique est complètement initialisé avant de le monter. |
| `Type=oneshot` | Le script s'exécute une seule fois et se termine. |
| `RemainAfterExit=yes` | Maintient le service "actif" après la fin du script afin que `ExecStop` soit déclenché lors de l'arrêt. |

---

## 4. Créer les règles udev

```bash
sudo tee /etc/udev/rules.d/99-local-usb-mount.rules > /dev/null << 'EOF'
# Montage automatique des partitions USB via systemd
KERNEL=="sd[a-z][0-9]*", SUBSYSTEMS=="usb", ACTION=="add", TAG+="systemd", ENV{SYSTEMD_WANTS}="usb-mount@%k.service"
# Lors du retrait, arrêter explicitement le service (SYSTEMD_WANTS ne déclenche pas ExecStop)
KERNEL=="sd[a-z][0-9]*", SUBSYSTEMS=="usb", ACTION=="remove", RUN+="/bin/systemctl stop usb-mount@%k.service"
EOF

```

Détail des règles :

| Filtre | Signification |
| --- | --- |
| `KERNEL=="sd[a-z][0-9]*"` | Correspond aux partitions de disques SCSI/USB : `sda1`, `sdb1`, `sda10`, etc. |
| `SUBSYSTEMS=="usb"` | Correspond uniquement aux périphériques connectés en USB (pas SATA, NVMe, etc.). |
| `ACTION=="add"` | Le périphérique a été branché. |
| `ACTION=="remove"` | Le périphérique a été débranché. |
| `TAG+="systemd"` | Indique à systemd de créer une unité de périphérique pour ce matériel. |
| `ENV{SYSTEMD_WANTS}` | Indique à systemd de démarrer le service nommé lorsque le périphérique apparaît. |
| `RUN+="/bin/systemctl stop ..."` | Lors du retrait, arrête explicitement le service (car `SYSTEMD_WANTS` déclenche uniquement les démarrages, pas les arrêts). |

> **Pourquoi ne pas utiliser `SYSTEMD_WANTS` pour le retrait ?** `SYSTEMD_WANTS` indique à systemd "quand ce périphérique apparaît, démarre aussi X". Lors du retrait, l'unité du périphérique disparaît — cela ne déclenche pas `ExecStop`. La directive `BindsTo=` gère la plupart des cas, mais le `RUN+=systemctl stop` est une approche de type "ceinture et bretelles" pour plus de fiabilité.

---

## 5. Tout activer

```bash
# Recharger les règles udev
sudo udevadm control --reload-rules
sudo udevadm trigger

# Recharger systemd pour prendre en compte le nouveau modèle de service
sudo systemctl daemon-reload

```

Aucun redémarrage n'est requis, mais un redémarrage ne fera pas de mal.

---

## 6. Tester

### Branchez une clé USB, puis vérifiez :

```bash
# Surveiller les journaux en temps réel (dans un terminal)
journalctl -t usb-mount -f

# Dans un autre terminal, vérifier les périphériques de type bloc
lsblk

# Vérifier le montage
ls /media/usb_sda1/

# Vérifier le statut du service
systemctl status usb-mount@sda1.service

```

### Débranchez la clé USB, puis vérifiez :

```bash
# Le point de montage devrait avoir disparu
ls /media/usb_*

# Le service devrait être inactif
systemctl status usb-mount@sda1.service

```

### Test manuel (sans rien brancher) :

```bash
# Simuler le montage (si /dev/sda1 existe)
sudo /usr/local/bin/usb-mount.sh add sda1

# Simuler le démontage
sudo /usr/local/bin/usb-mount.sh remove sda1

```

---

## 7. Dépannage

### Clé USB non détectée du tout

```bash
# Vérifier que le noyau voit le périphérique
dmesg | tail -20

# Lister tous les périphériques blocs
lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,TYPE,TRAN

```

### Le service ne démarre pas

```bash
# Vérifier si la règle udev se déclenche
udevadm monitor --property --subsystem-match=block

# Branchez ensuite la clé USB et cherchez SYSTEMD_WANTS dans la sortie

# Vérifier les journaux du service
journalctl -u 'usb-mount@*' --no-pager -n 50

```

### Le montage échoue silencieusement

```bash
# Vérifier les journaux pour trouver des erreurs
journalctl -t usb-mount --no-pager -n 20

# Essayer de monter manuellement pour voir l'erreur
sudo mount -v /dev/sda1 /mnt

```

### Erreurs courantes

| Symptôme | Cause | Solution |
| --- | --- | --- |
| `unknown filesystem type 'exfat'` | Prise en charge d'exFAT manquante | `sudo apt install exfat-fuse exfatprogs` |
| `unknown filesystem type 'ntfs'` | Prise en charge de NTFS manquante | `sudo apt install ntfs-3g` |
| `mount: permission denied` | Le script n'est pas exécutable | `sudo chmod +x /usr/local/bin/usb-mount.sh` |
| Le montage fonctionne manuellement mais pas en auto | Règles udev non chargées | `sudo udevadm control --reload-rules` |
| Le service reste actif après le débranchement | Anciennes règles sans `BindsTo` | Réappliquer les étapes 3 et 4 |

---

## 8. Comment ça marche

```
Clé USB branchée
       │
       ▼
   Le noyau crée /dev/sda1
       │
       ▼
   udev correspond à la règle (KERNEL=="sd[a-z][0-9]*", SUBSYSTEMS=="usb")
       │
       ▼
   udev définit ENV{SYSTEMD_WANTS}="usb-mount@sda1.service"
       │
       ▼
   systemd démarre usb-mount@sda1.service
       │
       ▼
   ExecStart s'exécute : /usr/local/bin/usb-mount.sh add sda1
       │
       ▼
   Le script détecte le système de fichiers (blkid), monte avec des options sécurisées
       │
       ▼
   Les fichiers sont disponibles dans /media/usb_sda1/


Clé USB débranchée
       │
       ▼
   Le noyau supprime /dev/sda1
       │
       ├──► udev RUN : systemctl stop usb-mount@sda1.service
       │
       └──► systemd BindsTo : dev-sda1.device a disparu → arrêt du service
                │
                ▼
          ExecStop : /usr/local/bin/usb-mount.sh remove sda1
                │
                ▼
          umount -l /media/usb_sda1 + rmdir

```

---

## 9. Notes de sécurité

Les options de montage sont choisies pour un contexte de **kiosque / station blanche** :

* **`noexec`** — Empêche l'exécution de tout binaire depuis la clé USB.
* **`nosuid`** — Empêche l'élévation de privilèges via setuid.
* **`nodev`** — Empêche les failles liées aux nœuds de périphériques.
* **`dmask=0022`** — Répertoires : propriétaire=rwx, groupe=r-x, autres=r-x (755).
* **`fmask=0133`** — Fichiers : propriétaire=rw-, groupe=r--, autres=r-- (644).
* **`uid=1001,gid=1001`** — Les fichiers appartiennent à l'utilisateur du kiosque (et non à root).

Pour une sécurité encore plus stricte, vous pourriez :

* Ajouter `ro` (lecture seule) à `COMMON_OPTS` si l'accès en écriture n'est pas nécessaire.
* Exécuter une analyse antivirus dans la fonction `do_mount()` après un montage réussi.
* Restreindre les ports USB avec des règles udev supplémentaires (par numéro de série, ID vendeur, etc.).

---

## 10. Désinstallation

Pour supprimer complètement la configuration de montage automatique :

```bash
# Supprimer les fichiers
sudo rm /usr/local/bin/usb-mount.sh
sudo rm /etc/systemd/system/usb-mount@.service
sudo rm /etc/udev/rules.d/99-local-usb-mount.rules

# Recharger
sudo udevadm control --reload-rules
sudo systemctl daemon-reload

# Nettoyer d'éventuels points de montage restants
sudo rmdir /media/usb_* 2>/dev/null

```

---

## Résumé des fichiers

| Fichier | Objectif |
| --- | --- |
| `/usr/local/bin/usb-mount.sh` | Script shell qui monte/démonte avec détection du système de fichiers |
| `/etc/systemd/system/usb-mount@.service` | Service modèle systemd (une instance par partition) |
| `/etc/udev/rules.d/99-local-usb-mount.rules` | Règles udev pour déclencher le service lors des événements USB |
