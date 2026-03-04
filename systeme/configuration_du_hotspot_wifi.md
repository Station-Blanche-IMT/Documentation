---
title: Hotspot Wi-Fi
layout: default
parent: Système
nav_order: 3
---

# Configuration du point d'accès Wi-Fi (Hotspot)

**Cible :** Raspberry Pi 5 / Raspberry Pi OS Bookworm
**Objectif :** Diffuser un réseau Wi-Fi depuis le Raspberry Pi via `nmcli` (NetworkManager), avec démarrage automatique.

---

## Prérequis

- NetworkManager actif (`nmcli` disponible).
- Interface Wi-Fi présente et reconnue (`wlan0`).
- Accès SSH ou Terminal.

---

## Étape 1 : Créer le hotspot

```bash
sudo nmcli device wifi hotspot ssid "BobberSecure" password "Bobber123!" ifname wlan0
```

| Paramètre | Valeur | Rôle |
|---|---|---|
| `ssid` | `BobberSecure` | Nom du réseau visible par les clients |
| `password` | `Bobber123!` | Mot de passe Wi-Fi |
| `ifname` | `wlan0` | Interface Wi-Fi à utiliser |

Cette commande crée un profil de connexion nommé `BobberSecure` et l'active immédiatement.

---

## Étape 2 : Activer le hotspot manuellement

Si le hotspot a été arrêté ou n'est pas actif :

```bash
sudo nmcli con up "BobberSecure"
```

---

## Étape 3 : Activer le démarrage automatique

Ces deux commandes configurent le hotspot pour qu'il se reconnecte automatiquement à chaque démarrage du système.

```bash
sudo nmcli connection modify "BobberSecure" connection.autoconnect yes
sudo nmcli connection modify "BobberSecure" connection.autoconnect-priority 100
```

> **`autoconnect-priority 100`** : Plus la valeur est élevée, plus cette connexion est choisie en priorité par NetworkManager si plusieurs profils Wi-Fi sont disponibles. La valeur par défaut est `0`.

---

## Étape 4 : Vérifier l'état des connexions

```bash
nmcli connection show
```

Résultat attendu :

```
NAME                 UUID                                  TYPE      DEVICE
BobberSecure         e3a8049a-94ce-4bfb-84d3-c53266b7f4fe  wifi      wlan0
Connexion filaire 1  06d254cb-784b-3107-a0fd-138cf5a2142c  ethernet  eth0
```

La colonne `DEVICE` doit afficher `wlan0` en face de `BobberSecure`, indiquant que le hotspot est actif.

---

## Option : Masquer le réseau (SSID caché)

Un SSID caché n'apparaît pas dans les listes Wi-Fi des appareils. Les clients doivent connaître le nom exact du réseau pour s'y connecter.

```bash
sudo nmcli connection modify "BobberSecure" 802-11-wireless.hidden yes
```

Relancer la connexion pour appliquer :

```bash
sudo nmcli con up "BobberSecure"
```

> **Note :** Un réseau caché n'est pas une mesure de sécurité forte — il reste détectable par des outils de scan réseau. Le combiner avec un mot de passe robuste reste indispensable.

---

## Récapitulatif des commandes

| Action                | Commande                                                                                |
| --------------------- | --------------------------------------------------------------------------------------- |
| Créer le hotspot      | `sudo nmcli device wifi hotspot ssid "BobberSecure" password "Bobber123!" ifname wlan0` |
| Activer manuellement  | `sudo nmcli con up "BobberSecure"`                                                      |
| Démarrage automatique | `sudo nmcli connection modify "BobberSecure" connection.autoconnect yes`                |
| Priorité de connexion | `sudo nmcli connection modify "BobberSecure" connection.autoconnect-priority 100`       |
| SSID caché            | `sudo nmcli connection modify "BobberSecure" 802-11-wireless.hidden yes`                |
| Vérifier l'état       | `nmcli connection show`                                                                 |
