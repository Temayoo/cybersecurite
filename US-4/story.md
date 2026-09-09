# US 4: Analyse dynamique

## Modifications système observées

une foisn`Env.exe` et `Res.exe` lancés,  les deux s'ajoutent aux
**Applications de démarrage** du Gestionnaire des tâches (persistance au
redémarrage). Cependant, seul `Res.exe` s'exécute réellement : `Env.exe`
crashe systématiquement avec l'erreur "could not find or load the Qt platform
plugin 'windows'" — il manque une DLL (`qwindows.dll`) absente de
l'échantillon fourni.

## Activités réseau identifiées

Aucune activité réseau détectée (Moniteur de ressources, onglet Réseau) 
pendant l'exécution.

## Création de fichiers / processus détectée

`Res.exe` se duplique à la racine du disque dans un dossier caché
`C:\WindSyst` (camouflage en faux dossier système), avec un fichier
`log.txt` qui enregistre les frappes clavier tapées pendant son exécution —
comportement de keylogger confirmé.

## Conclusion

Malware à deux composants : `Env.exe` (cassé — dépendance manquante)
et `Res.exe` (fonctionnel — persistance + keylogger local, pas
d'exfiltration réseau observée + Creation de clé de registre pour lancer l'application au démarrage ).