# Rapport d'investigation forensique et d'analyse de malware

## 1. Objectif

Ce rapport documente l'analyse de `Env.exe` et `Res.exe`, l'investigation de la machine virtuelle et l'identification de la clé USB associée. Il couvre les US 1 à 10 du backlog.

## 2. Conclusion

`Res.exe` est le composant actif observé. Il est présent dans `C:\WindSyst`, s'exécute depuis ce dossier, enregistre les frappes clavier dans `log.txt` et se lance automatiquement au démarrage de Windows. Il est également retrouvé en mémoire sous le PID 4504 et sur l'image disque.

`Env.exe` est détecté comme un trojan par VirusTotal. Son analyse statique montre des capacités réseau TLS, mais il ne démarre pas dans l'environnement fourni car le plugin Qt `windows` est absent. Aucune exfiltration réseau n'est donc démontrée.

La clé USB est identifiée comme une SanDisk Cruzer Micro, série `3514931B5ED0D062`. Les traces PnP de Windows établissent une première installation le 08/09/2026 à 10:41:09, une dernière connexion à 11:43:58 et un dernier retrait à 11:51:42.

Le risque global est **élevé**, en raison du keylogger local et de la persistance observée.

## 3. Périmètre et méthode

Les résultats reposent sur les fichiers `story.md` et les captures des dossiers `US-1` à `US-10`.

La méthode suit quatre étapes :

1. Isoler la VM avant d'exécuter les échantillons.
2. Identifier et analyser les fichiers sans les exécuter.
3. Observer les modifications dans la VM.
4. Recouper les traces mémoire, disque et USB.

Les observations et les déductions sont distinguées dans ce rapport. Une chaîne ou une fonction importée est un indice de capacité ; elle ne prouve pas une action réellement exécutée.

> Note de cohérence : l'US 1 mentionne Windows 11, tandis que l'US 6 mentionne Windows 10. Le rapport parle donc de « VM Windows ». La version exacte doit être harmonisée avant la présentation orale.

## 4. Éléments de preuve

### 4.1 Environnement et reconnaissance

La VM est configurée avec 8 Go de RAM, 4 CPU, 64 Go de stockage et un réseau Host Only. Cette configuration limite les communications de la VM au réseau privé de l'hôte.

| Élément | Résultat |
| --- | --- |
| Fichier reconnu | `Env.exe` |
| SHA-256 | `e09ec2098363a129de143fdaf73ad6e2e61266fba3f638a25214af3a8bc8f2f2` |
| Détection VirusTotal | 26 moteurs sur 71 lors de la consultation |
| Catégorie et labels | `trojan`, `keylogger`, `malgent` |

Preuves : `US-1/imageVM.png`, `US-2/imageVirusTotal.png`.

### 4.2 Analyse des exécutables

`Env.exe` et `Res.exe` sont des exécutables PE32 x86 construits avec Qt et MinGW. Aucune section typique d'UPX n'est relevée.

| Fichier | Éléments observés | Conclusion retenue |
| --- | --- | --- |
| `Env.exe` | `Qt5Network.dll`, `QSslSocket::connectToHostEncrypted`, `startClientEncryption`, `waitForEncrypted` | Capacités TLS ; aucune destination réseau en clair trouvée. |
| `Res.exe` | Chaînes `XCOPY` vers `C:\WindSyst`, DLL Qt/MinGW, référence à `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | Rôle probable d'installation et de persistance. |

Les indicateurs de compromission extraits sont `Env.exe`, `Res.exe`, `C:\WindSyst`, `C:\WindSyst\Env.exe`, `C:\WindSyst\Res.exe` et la clé de registre `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`.

### 4.3 Comportement observé dans la VM

| Observation | Résultat |
| --- | --- |
| Exécution de `Res.exe` | Fonctionnelle. |
| Exécution de `Env.exe` | Échec : plugin Qt `windows` / `qwindows.dll` manquant. |
| Fichiers créés | `Res.exe` copie les composants dans `C:\WindSyst` et crée `log.txt`. |
| Keylogger | Les frappes saisies pendant l'essai sont présentes dans `log.txt`. |
| Persistance | `Env` et `Res.exe` sont visibles dans les applications de démarrage. |
| Réseau | Aucune activité observée pendant l'essai. |

Preuves : `US-4/imageWindSyst.png`, `US-4/imageLog.png`, `US-4/imageAppDemarrage.png`, `US-4/imageActiviteReseau.png`.

### 4.4 Analyse mémoire

Le dump `memory-dump.elf` est généré avec `VBoxManage debugvm "Windows" dumpvmcore`. Sa taille documentée est de 2 287 859 496 octets. Il est lu avec Volatility 3.

| Élément | Résultat |
| --- | --- |
| Processus | `Res.exe` |
| PID / PPID | 4504 / 2712 |
| Architecture | 32 bits sous Windows 64 bits (`Wow64=True`) |
| Arbre de processus | `userinit.exe` > `explorer.exe` > `Res.exe` > `conhost.exe` |
| Ligne de commande | `C:\WindSyst\Res.exe` |
| Connexion réseau pour le PID 4504 | Aucune dans `windows.netscan` |
| Résultat de `malfind` | Aucune région remontée pour ce processus |

Les DLL Qt et MinGW sont chargées depuis `C:\WindSyst`. La présence de `WS2_32.dll` ne prouve pas une connexion réseau active.

Preuves : `US-6/01-running-vm.png` à `US-6/08-dlllist.png`.

### 4.5 Analyse disque et corrélation

Le fichier `Windows.vdi` est copié vers `Windows-forensic.vdi`. Les deux empreintes SHA-256 sont identiques :

```text
63D03BE07129EEFA538706C376C3640D634AD6A732F608D5735B968D86E64E6F
```

Une copie de travail au format VHD est montée en lecture seule. Elle contient `C:\WindSyst\Res.exe`, soit le même exécutable que celui retrouvé en mémoire.

| Mémoire | Disque | Conclusion |
| --- | --- | --- |
| `Res.exe`, PID 4504 | `C:\WindSyst\Res.exe` | Le processus correspond au fichier analysé. |
| DLL Qt et MinGW chargées depuis `C:\WindSyst` | Fichiers associés dans le même dossier | Environnement d'exécution cohérent. |
| Lancement automatique observé | Aucune référence directe dans les emplacements vérifiés | Persistance observée, mécanisme exact non identifié. |

Les clés HKLM Run et RunOnce, les dossiers Startup et les tâches planifiées sont contrôlés sans référence à `Res.exe` ni à `WindSyst`. La prochaine vérification utile est la ruche utilisateur `NTUSER.DAT`, notamment `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`.

Preuves : `US-7/03-hash-comparison.png`, `US-7/05-res-present.png`, `US-8/01-recherche-persistance.png`.

### 4.6 Identification et chronologie USB

L'US 9 identifie la clé par `lsblk`. L'US 10 retrouve le même numéro de série dans les informations PnP de Windows.

| Information | Valeur |
| --- | --- |
| Fabricant / modèle | SanDisk Cruzer Micro |
| Numéro de série | `3514931B5ED0D062` |
| VID / PID | `0781` / `5151` |
| Lettre de lecteur observée | `E:\` |

| Événement | Horodatage |
| --- | --- |
| Première installation | 08/09/2026 10:41:09 |
| Dernière connexion | 08/09/2026 11:43:58 |
| Dernier retrait | 08/09/2026 11:51:42 |

La dernière connexion documentée dure au plus 7 minutes et 44 secondes avant le retrait. Ces traces prouvent l'utilisation du périphérique dans Windows, mais pas les fichiers éventuellement copiés pendant cette période.

Preuve : `US-10/01-historique-usb.png`.

## 5. Couverture du backlog

| US | Critères d'acceptation | Résultat |
| --- | --- | --- |
| US 1 | VM isolée ; réseau contrôlé | VM en Host Only. |
| US 2 | Hash ; détection ; base existante | SHA-256 et résultat VirusTotal documentés. |
| US 3 | Fonctions ; chaînes/imports/sections ; IoC | Analyse statique et IoC extraits. |
| US 4 | Modifications ; réseau ; fichiers/processus | Keylogger et fichiers observés ; aucun trafic réseau observé. |
| US 5 | Impacts ; risque ; mitigations | Risque élevé et mesures formulées. |
| US 6 | Commandes ; dump exploitable ; processus ; réseau | Dump analysé avec Volatility ; `Res.exe` identifié ; pas de connexion pour le PID 4504. |
| US 7 | Image ; outils ; fichier suspect | Copie VDI vérifiée par hash ; `Res.exe` retrouvé. |
| US 8 | Correspondance RAM/disque ; persistance | Corrélation confirmée ; mécanisme de persistance non localisé. |
| US 9 | Numéro de série extrait | Clé SanDisk, série `3514931B5ED0D062`. |
| US 10 | Horodatages ; chronologie | Première installation, dernière connexion et retrait établis. |

## 6. Scénario reconstitué

1. `Res.exe` prépare ou copie les composants dans `C:\WindSyst`.
2. Il s'exécute depuis ce dossier avec ses DLL Qt et MinGW.
3. Il enregistre les frappes clavier dans `log.txt`.
4. Son lancement automatique est observé au démarrage de Windows.
5. Le même exécutable est retrouvé dans le dump mémoire et sur l'image disque.
6. La clé SanDisk de série `3514931B5ED0D062` est utilisée sur Windows le 08/09/2026.

Ce scénario ne permet pas de conclure à une exfiltration réseau, d'identifier le mécanisme exact de persistance, ni de déterminer quels fichiers ont été copiés vers la clé USB.

## 7. Risque et recommandations

| Domaine | Niveau | Justification |
| --- | --- | --- |
| Confidentialité | Élevé | Les frappes sont enregistrées localement. |
| Persistance | Élevé | Le lancement automatique est observé. |
| Réseau | Non démontré | Aucun trafic n'est observé et `Env.exe` ne démarre pas. |

Mesures recommandées :

- Maintenir l'antivirus ou EDR et la protection contre les falsifications actifs.
- Désactiver AutoRun et AutoPlay sur les postes concernés.
- Restreindre et tracer l'usage des supports USB.
- Contrôler les emplacements de démarrage de Windows.
- En cas de compromission, isoler le poste, changer les secrets exposés et réinstaller le système si son intégrité ne peut pas être garantie.

## 8. Limites et suites

- `Env.exe` ne démarre pas avec l'échantillon fourni : aucune exfiltration ne peut être attribuée au malware sur la base de ces essais.
- Le lancement automatique de `Res.exe` est observé, mais son mécanisme n'est pas localisé.
- La chronologie USB ne permet pas de prouver les fichiers transférés. Les journaux Windows et les artefacts utilisateur doivent être vérifiés pour compléter ce point.
