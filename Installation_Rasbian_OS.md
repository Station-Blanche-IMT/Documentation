# Installation de DebianOS sur raspberry Pi 5
##  Utilisation de la carte SD pour boot dessus.

## Téléchargé cette image : 
```bash
wget https://downloads.raspberrypi.com/raspios_arm64/images/raspios_arm64-2025-05-13/2025-05-13-raspios-bookworm-arm64.img.xz
```

## Décompresser l'image télécharger
```bash
xz -d 2025-05-13-raspios-bookworm-arm64.img.xz
```

## On obtiens :
```bash
2025-05-13-raspios-bookworm-arm64.img
```

## On cherche le nom du disque nvme
```bash
lsblk
```

nvme0n1     259:0    0 476,9G  0 disk
├─nvme0n1p1 259:1    0   512M  0 part /boot/firmware
└─nvme0n1p2 259:2    0 476,4G  0 part /

## On copie l'iamge OS sur le disque nvme
```bash
sudo dd if=2025-05-13-raspios-bookworm-arm64.img of=/dev/nvme0n1 bs=4M status=progress
```

## Synchronisation
```bash
sync
```

## On change l'ordre de boot 
```bash
raspi-config
```

## Ensuite suivre ce chemin et selectionner le nvme comme device de boot principale :
```bash
6 Advanced Options -> A4 Boot Order -> B2 NVMe/USB Boot Boot from NVMe before trying USB and then SD Card
```