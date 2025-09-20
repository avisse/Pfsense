# Runbook d'exploitation — pfSense

## Sauvegarde / Restauration
- Sauvegarde: Diagnostics → Backup/Restore → Download configuration as XML
- Restaurer: Diagnostics → Backup/Restore → Upload
- Ne jamais publier `config.xml` dans un dépôt public (contient des secrets).

## Mises à jour
- System → Update → vérifier la branche et appliquer les patchs.
- Redémarrage planifié en créneau de maintenance.

## Ajout / modification de règle
1) Créer des aliases (host/ports) si nécessaire.
2) Firewall → Rules → [Interface] → Add.
3) Renseigner action/source/destination/protocol/ports/description.
4) Positionner la règle (drag & drop) au bon endroit.
5) Sauvegarder → Apply Changes.
6) Tester et vérifier les logs.

## Vérification de l’ordre des règles
- Les règles sont évaluées **de haut en bas**.
- Les règles de blocage spécifiques doivent être **au-dessus** des passes génériques.

## Journalisation
- Status → System Logs → Firewall
- Diagnostics → Packet Capture (debug) puis **Download capture** pour analyse Wireshark.

## Sécurité
- Changer le mot de passe admin, restreindre l’accès GUI au LAN uniquement.
- Désactiver SSH si inutile, ou restreindre par clé/règles.
- Activer NTP et synchroniser les horloges.
