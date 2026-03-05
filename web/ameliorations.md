---
title: Pistes d'amélioration
layout: default
parent: Application Web
nav_order: 8
---

# Pistes d'amélioration
{: .no_toc }

Axes de travail suggérés pour les futures équipes.

## Table des matières
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Améliorations suggérées

1. **Nettoyage automatique du staging** : Implémenter un mécanisme (middleware ou tâche périodique) pour supprimer les dossiers orphelins dans `/var/lib/station-blanche/staging/`

2. **Pagination des logs** : Actuellement limités à 200/500 entrées — ajouter une pagination côté serveur et des filtres (par date, catégorie, niveau)

3. **Export des logs** : Permettre l'export CSV/PDF des journaux depuis l'interface admin

4. **Timeout de session** : Configurer un timeout de session automatique pour éviter qu'un utilisateur reste connecté indéfiniment

5. **Multi-badges** : Optimiser la vérification des badges (actuellement en O(n) sur tous les badges) — potentiellement avec un index ou une table de lookup

6. **Tests automatisés** : Écrire des tests unitaires et d'intégration (`sb_app/tests.py` est vide)

7. **Notifications** : Alerter l'administrateur quand la base ClamAV est trop ancienne (>30 jours par exemple)

8. **Internationalisation** : Le code est en français, ajouter le support multi-langue via Django i18n si nécessaire

9. **HTTPS** : Configurer Apache avec un certificat SSL/TLS si la station est accessible sur un réseau

10. **Mise à jour ClamAV en ligne** : Si la station a accès à Internet dans le futur, intégrer `freshclam`

11. **Gestion des erreurs USB** : Améliorer la gestion des cas limites (clé USB retirée pendant la copie, etc.)

12. **Clavier virtuel amélioré** : Ajouter les chiffres, caractères spéciaux et majuscules au clavier virtuel d'ajout de badge

---

## Annexe : Commandes utiles

```bash
# Vérifier le statut des services
sudo systemctl status station-blanche-web
sudo systemctl status apache2

# Voir les logs Gunicorn
sudo journalctl -u station-blanche-web -f

# Voir les logs Apache
sudo tail -f /var/log/apache2/station-blanche-*.log

# Appliquer les migrations manuellement
cd /home/kiosk/station-blanche-web/sb_proj
source ../.venv/bin/activate
python3 manage.py migrate

# Accéder au shell Django
python3 manage.py shell

# Lister les badges en BDD
python3 manage.py shell -c "from sb_app.models import Badge; print(list(Badge.objects.values_list('name', flat=True)))"

# Vérifier les fichiers ClamAV
ls -la clamav_database/

# Redémarrer les services après modification du code
sudo systemctl restart station-blanche-web
sudo systemctl restart apache2
```
