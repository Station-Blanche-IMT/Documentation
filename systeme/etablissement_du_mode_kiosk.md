---
title: Mode Kiosk
layout: default
parent: Système
nav_order: 2
---

# Installation Mode Kiosk — Raspberry Pi (Station Blanche)

**Cible :** Raspberry Pi 5 / Raspberry Pi OS Bookworm (Debian 13)
**Objectif :** Démarrer automatiquement sur une page web en plein écran, sans bureau, sécurisé, sous X11.

---

## Prérequis

- Installation fraîche de Raspberry Pi OS (Bookworm).
- Accès Internet pour le téléchargement des paquets.
- Accès SSH ou Terminal.

---

## Étape 0 : Préparation du système et de l'utilisateur

### 0.1 — Mettre à jour le système et installer les dépendances

On installe `openbox` (gestionnaire de fenêtres minimal — écran noir par défaut), `unclutter` (cache le curseur de souris), et `chromium` (le navigateur).

> **Note :** Sur Raspberry Pi OS Bookworm, le paquet s'appelle `chromium` et non `chromium-browser`.

```bash
sudo apt update
sudo apt install openbox unclutter chromium -y
```

### 0.2 — Créer l'utilisateur dédié "kiosk"

```bash
sudo adduser kiosk
```

Suivez les instructions (le mot de passe importe peu).

> **Sécurité :** Ne **pas** ajouter cet utilisateur au groupe `sudo`. Il ne doit avoir aucun droit d'administration générale.
> Si l'utilisateur a déjà été ajouté au groupe sudo par erreur, le retirer :

```bash
sudo deluser kiosk sudo
```

> L'utilisateur `kiosk` aura uniquement le droit d'exécuter `poweroff` et `reboot` sans mot de passe, via une règle ciblée dans `/etc/sudoers.d/` (voir Étape 4.3).

---

## Étape 1 : Basculer le Raspberry Pi sous X11

Par défaut, le Pi 5 sous Bookworm utilise Wayland, qui est encore instable pour un mode Kiosk strict. On bascule sur X11.

```bash
sudo raspi-config
```

1. Allez dans **6 Advanced Options** → **A6 Wayland**.
2. Choisissez **W1 X11**.
3. Le système demandera de redémarrer. **Faites-le.**

**Vérification après redémarrage :**

```bash
loginctl show-session $(loginctl list-sessions --no-legend | awk '{print $1}' | head -1) -p Type
```

Doit afficher : `Type=x11`

---

## Étape 2 : Configuration de l'Autologin (LightDM)

On force l'écran de connexion (LightDM) à connecter automatiquement l'utilisateur `kiosk` sur une session personnalisée.

### 2.1 — Éditer la configuration de LightDM

```bash
sudo nano /etc/lightdm/lightdm.conf
```

Chercher la section `[Seat:*]` (vers la fin du fichier). Modifier ou ajouter les trois lignes suivantes (décommenter si elles existent avec un `#`) :

```ini
[Seat:*]
# ... (ne pas toucher aux autres lignes existantes) ...
autologin-user=kiosk
autologin-session=kiosk
user-session=kiosk
```

> **Important :** S'assurer qu'il n'y a pas d'espace autour du `=`.

### 2.2 — Créer la définition de la session "Kiosk"

Ce fichier indique au système où trouver le script de démarrage pour la session `kiosk`.

```bash
sudo nano /usr/share/xsessions/kiosk.desktop
```

Coller exactement ce contenu :

```ini
[Desktop Entry]
Name=Kiosk
Comment=Kiosk Mode
Exec=/home/kiosk/.xsession
Type=Application
```

---

## Étape 3 : Création des scripts de démarrage

Tout se passe dans le dossier `/home/kiosk/`.

### A — Le fichier `.xsession` (Le chef d'orchestre)

Ce fichier remplace le chargement du bureau classique. Il initialise l'écran et lance le navigateur.

```bash
sudo nano /home/kiosk/.xsession
```

Coller le contenu suivant :

```bash
#!/bin/bash

# 1. Gestion écran (Empêcher la mise en veille et l'écran noir)
xset s off
xset s noblank
xset -dpms

# 2. Lancer le gestionnaire de fenêtres (INDISPENSABLE pour le plein écran)
openbox-session &

# 3. Masquer la souris après 2 secondes d'inactivité
unclutter -idle 2 &

# 4. Boucle infinie : Si Chromium crash ou est fermé, il se relance
while true; do
  /home/kiosk/kiosk.sh
  sleep 2
done
```

### B — Le fichier `kiosk.sh` (Le navigateur)

Ce script contient la commande Chromium avec toutes les sécurités activées.

```bash
sudo nano /home/kiosk/kiosk.sh
```

Coller le contenu suivant :

```bash
#!/bin/bash

# Nettoyage préventif pour éviter la bulle "Chromium ne s'est pas fermé correctement"
sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' ~/.config/chromium/'Local State'
sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' ~/.config/chromium/Default/Preferences

# Attendre que le serveur web soit prêt
echo "En attente du serveur sur 127.0.0.1..."
until curl -s -o /dev/null -w '' http://127.0.0.1/ 2>/dev/null; do
    sleep 1
done
echo "Serveur prêt, lancement de Chromium."

# Lancement de Chromium avec les flags de sécurité
# --password-store=basic   : Supprime la demande de mot de passe (Keyring)
# --no-first-run           : Supprime l'écran de bienvenue
# --kiosk                  : Mode plein écran strict (pas de barre d'adresse)
# --incognito              : Navigation privée (pas de cache/historique persistant)
# --noerrdialogs           : Masque les dialogues d'erreur
# --disable-infobars       : Masque les barres d'information
# --overscroll-history-navigation=0  : Bloque le "Swipe Back" tactile
# --disable-pinch          : Désactive le zoom par pincement
# --check-for-update-interval       : Désactive les vérifications de mise à jour
# --disable-features       : Désactive le swipe de navigation (double sécurité)

chromium \
  --password-store=basic \
  --no-first-run \
  --kiosk \
  --incognito \
  --noerrdialogs \
  --disable-infobars \
  --overscroll-history-navigation=0 \
  --disable-pinch \
  --check-for-update-interval=31536000 \
  --disable-features=TouchpadOverscrollHistoryNavigation,OverscrollHistoryNavigation \
  http://127.0.0.1
```

> **Note :** Adapter l'URL (`http://127.0.0.1`) à votre configuration. Si votre serveur tourne sur un autre port (ex: Django sur 8000), remplacer par `http://127.0.0.1:8000` et ajuster la commande `curl` de la même façon.

---

## Étape 4 : Permissions et nettoyage

### 4.1 — Rendre les scripts exécutables et les attribuer à l'utilisateur kiosk

```bash
sudo chmod +x /home/kiosk/.xsession
sudo chmod +x /home/kiosk/kiosk.sh
sudo chown kiosk:kiosk /home/kiosk/.xsession
sudo chown kiosk:kiosk /home/kiosk/kiosk.sh
```

### 4.2 — Supprimer le trousseau de clés (au cas où il existerait)

Cela évite que Chromium demande un mot de passe de déverrouillage au démarrage.

```bash
sudo rm -rf /home/kiosk/.local/share/keyrings/
```

### 4.3 — Autoriser l'utilisateur kiosk à éteindre / redémarrer sans mot de passe

L'interface web Django pilote l'arrêt et le redémarrage via `subprocess.Popen(["sudo", "systemctl", "poweroff/reboot"])`. Django tournant en tâche de fond sans terminal interactif, il est impossible de saisir un mot de passe : il faut une règle `NOPASSWD` ciblée.

> **Pourquoi un fichier dédié dans `sudoers.d/` et non dans `sudoers` ?**
> Les fichiers dans `/etc/sudoers.d/` sont lus par ordre alphabétique et **la dernière règle correspondante l'emporte**. Un fichier préfixé `020_` est lu après le fichier `010_pi-nopasswd` (s'il existe), garantissant que la règle `NOPASSWD` n'est pas écrasée.

Créer le fichier avec `visudo` (valide la syntaxe avant de sauvegarder) :

```bash
sudo visudo -f /etc/sudoers.d/020_kiosk-power
```

Coller le contenu suivant :

```
kiosk ALL=(ALL) NOPASSWD: /usr/bin/systemctl poweroff, /usr/bin/systemctl reboot
```

Appliquer les permissions obligatoires (sudoers refuse les fichiers lisibles en écriture) :

```bash
sudo chmod 0440 /etc/sudoers.d/020_kiosk-power
```

**Vérification :**

```bash
sudo -n -l /usr/bin/systemctl reboot
sudo -n -l /usr/bin/systemctl poweroff
```

Si le code de retour est `0` (aucune demande de mot de passe), la règle est active.

### 4.4 — Redémarrer pour appliquer

```bash
sudo reboot
```

---

## Étape 5 : Bloquer le swipe de navigation dans Chromium (écran tactile)

Sur un écran tactile, un glissement horizontal déclenchait la navigation arrière/avant du navigateur. Trois niveaux de protection ont été mis en place.

### 5.1 — Politique d'entreprise Chromium (niveau le plus fiable)

Cette méthode applique la restriction au niveau du navigateur lui-même, indépendamment des flags de la ligne de commande.

```bash
sudo mkdir -p /etc/chromium/policies/managed
sudo nano /etc/chromium/policies/managed/kiosk.json
```

Coller le contenu suivant :

```json
{
  "OverscrollHistoryNavigationEnabled": false,
  "TranslateEnabled": false
}
```

**Vérification :** Ouvrir `chrome://policy` dans Chromium. La ligne `OverscrollHistoryNavigationEnabled` doit apparaître avec la valeur `false`.

### 5.2 — Flags Chromium dans `kiosk.sh` (déjà configuré)

Les flags suivants sont déjà présents dans `/home/kiosk/kiosk.sh` :

```bash
--overscroll-history-navigation=0 \
--disable-features=TouchpadOverscrollHistoryNavigation,OverscrollHistoryNavigation
```

| Flag | Cible |
|------|-------|
| `--overscroll-history-navigation=0` | Désactive le swipe (générique) |
| `TouchpadOverscrollHistoryNavigation` | Désactive pour les touchpads |
| `OverscrollHistoryNavigation` | Désactive pour les écrans tactiles |

### 5.3 — CSS dans l'application web (défense en profondeur)

Dans le template HTML principal de l'application (ex: `base.html`), ajouter :

```css
html {
    overscroll-behavior-x: none;
}

body {
    overscroll-behavior-x: none;
    touch-action: pan-y pinch-zoom;
}
```

- `overscroll-behavior-x: none` — empêche le navigateur d'interpréter un défilement horizontal excessif comme une navigation.
- `touch-action: pan-y pinch-zoom` — limite les gestes tactiles au défilement vertical et au zoom, bloquant le balayage horizontal.

---

## Résultat attendu

Au redémarrage :

1. L'écran s'allume.
2. Bref écran noir (chargement d'Openbox).
3. Si le serveur web n'est pas encore prêt, le script attend en boucle.
4. Chromium s'affiche directement en plein écran sur l'URL configurée.
5. Pas de curseur de souris visible (masqué après 2s d'inactivité).
6. Pas de zoom tactile, pas de retour arrière par glissement.
7. Aucune demande de mot de passe (keyring/trousseau).
8. Si Chromium crash ou se ferme, il se relance automatiquement sous 2 secondes.

---

## Récapitulatif des fichiers

| Fichier | Rôle |
|---|---|
| `/etc/lightdm/lightdm.conf` | Autologin de l'utilisateur `kiosk` sur la session `kiosk` |
| `/usr/share/xsessions/kiosk.desktop` | Définition de la session X → pointe vers `.xsession` |
| `/home/kiosk/.xsession` | Chef d'orchestre : écran, openbox, souris, boucle de relance |
| `/home/kiosk/kiosk.sh` | Lance Chromium en mode kiosk avec tous les flags de sécurité |
| `/etc/sudoers.d/020_kiosk-power` | Autorise `kiosk` à exécuter `poweroff`/`reboot` sans mot de passe |
| `/etc/chromium/policies/managed/kiosk.json` | Politique Chromium : désactive navigation swipe et traduction |

---

## Dépannage

### Chromium affiche "Chromium ne s'est pas fermé correctement"
Les commandes `sed` dans `kiosk.sh` nettoient ce problème automatiquement. Si le problème persiste, supprimer manuellement le profil :
```bash
rm -rf /home/kiosk/.config/chromium/
```

### L'écran reste noir
- Vérifier que X11 est bien actif : `loginctl show-session ... -p Type` doit afficher `x11`.
- Vérifier que `openbox` est installé : `dpkg -l openbox`.
- Vérifier les logs : `cat /var/log/lightdm/lightdm.log`.

### Chromium ne se lance pas en plein écran
- `openbox-session` doit être lancé **avant** Chromium (c'est le `&` dans `.xsession`).
- Sans gestionnaire de fenêtres, le flag `--kiosk` ne peut pas fonctionner.

### Demande de mot de passe "Keyring" au démarrage
- Le flag `--password-store=basic` dans `kiosk.sh` résout ce problème.
- Supprimer également le dossier keyrings : `rm -rf /home/kiosk/.local/share/keyrings/`.

### Les boutons Éteindre / Redémarrer de l'interface web ne fonctionnent pas
- Vérifier que le fichier `/etc/sudoers.d/020_kiosk-power` existe et a les bonnes permissions (`0440`).
- Tester sans mot de passe : `sudo -n -l /usr/bin/systemctl reboot` (doit retourner `0`).
- S'assurer qu'aucun fichier dans `sudoers.d/` avec un préfixe alphabétiquement supérieur à `020_` ne réécrit la règle avec une exigence de mot de passe.

### Le swipe tactile déclenche toujours la navigation arrière/avant
- Vérifier la politique Chromium : ouvrir `chrome://policy` et confirmer `OverscrollHistoryNavigationEnabled = false`.
- Si la politique n'apparaît pas, vérifier que le fichier `/etc/chromium/policies/managed/kiosk.json` est syntaxiquement valide : `python3 -m json.tool /etc/chromium/policies/managed/kiosk.json`.
- Vérifier que les flags `--overscroll-history-navigation=0` et `--disable-features=...OverscrollHistoryNavigation` sont bien présents dans `kiosk.sh`.