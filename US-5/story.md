# US 5: Évaluation des risques

## Description claire des impacts

- **Confidentialité (impact fort)** : keylogger actif capturant les frappes
  clavier en clair (mots de passe, messages, données sensibles) dans
  `C:\WindSyst\log.txt`.
- **Persistance** : `Env.exe` et `Res.exe` s'enregistrent dans les
  Applications de démarrage — le malware survit à un redémarrage tant qu'il
  n'est pas manuellement supprimé.
- **Camouflage** : dossier `C:\WindSyst` imitant un nom de dossier système
  légitime, rendant la détection manuelle plus difficile pour un utilisateur
  non averti.
- **Vecteur d'infection** : propagation par clé USB (US9/US10), donc capable
  de contourner les défenses réseau périmétriques.
- **Exfiltration** : `Env.exe` est censé gérer la partie réseau (probablement
  pour envoyer `log.txt` à l'attaquant, vu la présence de `Qt5Network.dll`),
  mais il ne se lance jamais faute de dépendances manquantes. Dans cet état,
  les logs restent bloqués localement sur la machine.

## Niveau de risque évalué

**Élevé.**  Justification : vol potentiel d'identifiants/données sensibles
(keylogger) + persistance confirmée + vecteur USB contournant le réseau.
Facteur atténuant : signature déjà connue et détectée par Windows Defender
(`Trojan:Win32/Malgent!MSR`) — un poste avec antivirus actif et protection
contre les falsifications activée aurait bloqué l'infection sans
intervention manuelle.

## Recommandations de sécurité formulées

- Antivirus/EDR actif + protection contre les falsifications activée en
  permanence.
- Désactiver l'AutoRun/AutoPlay des clés USB.
- Contrôler les périphériques USB autorisés sur les postes sensibles.
- Surveiller les clés de démarrage (Run/Startup) pour repérer une
  persistance suspecte.
- Si compromission confirmée : changer les mots de passe, supprimer
  `C:\WindSyst`, ou ré-imager la machine.