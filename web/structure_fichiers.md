---
title: Structure des fichiers
layout: default
parent: Application Web
nav_order: 2
---

# Structure des fichiers
{: .no_toc }

Arborescence complète du projet Django.

---

```
station-blanche-web/
├── README.md                     # Présentation rapide
├── DOCUMENTATION.md              # Documentation complète
├── requirements.txt              # Dépendances Python
├── Start_App.sh                  # Script de déploiement production
├── station-blanche-web.conf      # Configuration Apache2 (VirtualHost)
├── station-blanche-web.service   # Service systemd (Gunicorn)
├── error-pages/
│   └── error.html                # Page d'erreur Apache personnalisée
│
└── sb_proj/                      # Racine du projet Django
    ├── manage.py                 # CLI Django
    ├── db.sqlite3                # Base de données SQLite
    ├── clamav_script.py          # Moteur de scan ClamAV
    ├── script_update_clamav.py   # MAJ base antivirale depuis clé USB
    ├── scripts.py                # Utilitaires (USB, copie, nettoyage)
    │
    ├── clamav_database/          # Base de signatures ClamAV (hors-ligne)
    │   ├── main.cvd
    │   ├── daily.cvd
    │   └── bytecode.cvd
    │
    ├── sb_proj/                  # Configuration Django
    │   ├── settings.py           # Paramètres (BDD, timezone, etc.)
    │   ├── urls.py               # Routes racine + handler500
    │   ├── wsgi.py               # Point d'entrée WSGI
    │   └── asgi.py               # Point d'entrée ASGI (non utilisé)
    │
    └── sb_app/                   # Application principale
        ├── models.py             # Modèles : Badge, LogEntry
        ├── views.py              # Vues (toute la logique métier)
        ├── urls.py               # Routes de l'application
        ├── admin.py              # Interface d'admin Django
        ├── apps.py               # Configuration de l'app
        │
        ├── config/
        │   └── config.json       # Configuration dynamique (mode, PIN, MAJ ClamAV)
        │
        ├── logs/                 # Fichiers CSV de scan + logs Django
        │   ├── admin.csv
        │   ├── login.csv
        │   ├── clamav.csv
        │   └── scan-YYYYMMDD-HHMMSS.csv
        │
        ├── migrations/           # Migrations Django
        │   ├── 0001_initial.py
        │   └── 0002_badge_unique_name.py
        │
        └── templates/
            ├── base.html                      # Template de base (header, nav, CSS, icônes SVG)
            ├── 500.html                       # Page erreur 500
            └── sb_app/
                ├── index.html                 # Page d'accueil (liste USB)
                ├── standard_user_login.html   # Page de scan badge
                ├── entree_pin.html            # Saisie PIN admin
                ├── config.html                # (non utilisé actuellement)
                ├── change_pin.html            # Changement PIN
                ├── error.html                 # Page d'erreur applicative
                ├── logs.html                  # Affichage logs (utilisateur)
                ├── protected.html             # Hub admin (6 tuiles)
                ├── target_copy_midstep.html   # Choix clé cible / serveur
                ├── target_copy_end.html       # Résultat copie
                ├── virus_action.html          # Action sur fichiers infectés
                ├── usb_cleanup_midstep.html   # Étape intermédiaire nettoyage
                ├── usb_cleanup_end.html       # Résultat nettoyage
                └── protected/
                    ├── protected_cards.html       # Hub gestion badges
                    ├── protected_cards_add.html   # Ajout badge (2 étapes)
                    ├── protected_cards_list.html   # Liste des badges
                    ├── protected_cards_remove.html # Suppression badge par scan
                    ├── protected_clamav.html      # MAJ ClamAV
                    ├── protected_logs.html         # Journaux admin
                    ├── protected_pin.html          # Changement PIN
                    ├── protected_power.html        # Arrêt/Redémarrage
                    └── protected_security.html     # Mode de sécurité
```
