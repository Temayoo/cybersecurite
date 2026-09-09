# US 3 : Analyse statique

L'objectif de cette analyse est d'étudier le malware sans l'exécuter afin d'identifier sa structure, ses dépendances, ses fonctionnalités principales et les indicateurs de compromission associés.

## 1. Identification des fichiers

La commande suivante a été utilisée :

```bash
file *
```

L'analyse montre que le malware est principalement constitué de deux exécutables Windows :

```text
Env.exe : PE32 executable for MS Windows, GUI, Intel i386, 8 sections
Res.exe : PE32 executable for MS Windows, console, Intel i386, 8 sections
```

Plusieurs bibliothèques sont également présentes :

```text
Qt5Core.dll
Qt5Gui.dll
Qt5Network.dll
Qt5Widgets.dll
libgcc_s_dw2-1.dll
libstdc++-6.dll
libwinpthread-1.dll
```

Les deux exécutables sont donc des programmes Windows **32 bits x86**.

La présence des bibliothèques Qt ainsi que des bibliothèques GCC indique que les exécutables ont probablement été développés en **C++ avec Qt et compilés avec MinGW**.

---

## 2. Analyse des sections

La commande suivante a été utilisée :

```bash
objdump -h Env.exe
```

`Env.exe` contient les sections suivantes :

| Section    | Rôle                       |
| ---------- | -------------------------- |
| `.text`    | code exécutable            |
| `.data`    | données initialisées       |
| `.rdata`   | données en lecture seule   |
| `.eh_fram` | gestion des exceptions C++ |
| `.bss`     | données non initialisées   |
| `.idata`   | table des imports          |
| `.CRT`     | données du runtime C/C++   |
| `.tls`     | stockage local aux threads |

Les sections correspondent globalement à celles d'un exécutable Windows classique compilé avec GCC/MinGW.

Aucune section typique d'un packer comme `UPX0` ou `UPX1` n'a été observée.

Les en-têtes de `Env.exe` indiquent également :

```text
Format : PE32
Architecture : 32 bits
Subsystem : Windows GUI
Entry Point : 0x14c0
Image Base : 0x00400000
Date de compilation indiquée : 22/12/2017 01:40:43
```

Les symboles et les informations de débogage ont été supprimés de l'exécutable.

---

## 3. Analyse des imports de Env.exe

Les principales DLL importées par `Env.exe` sont :

```text
Qt5Core.dll
Qt5Network.dll
Qt5Widgets.dll
libgcc_s_dw2-1.dll
KERNEL32.dll
msvcrt.dll
SHELL32.dll
libstdc++-6.dll
```

### Fonctions réseau

L'import de `Qt5Network.dll` met en évidence plusieurs fonctions liées aux communications réseau :

```text
QSslSocket::waitForEncrypted
QSslSocket::startClientEncryption
QSslSocket::connectToHostEncrypted
QSslSocket
```

Ces fonctions montrent que `Env.exe` dispose de capacités permettant d'établir une **connexion réseau chiffrée via SSL/TLS**.

L'utilisation de :

```text
connectToHostEncrypted
startClientEncryption
waitForEncrypted
```

suggère que l'exécutable peut communiquer avec un serveur distant via une connexion chiffrée.

### Fonctions Windows

Plusieurs fonctions de `KERNEL32.dll` sont également importées :

```text
GetCurrentProcess
GetCurrentProcessId
GetCurrentThreadId
GetProcAddress
LoadLibraryA
Sleep
TerminateProcess
VirtualProtect
VirtualQuery
```

`GetProcAddress` et `LoadLibraryA` permettent notamment de charger dynamiquement des bibliothèques et de résoudre des fonctions à l'exécution.

`VirtualProtect` permet de modifier les protections associées à des zones mémoire.

Ces fonctions ne sont pas malveillantes en elles-mêmes, mais peuvent être intéressantes dans le contexte d'un malware.

---

## 4. Analyse des imports de Res.exe

`Res.exe` importe notamment :

```text
Qt5Core.dll
libgcc_s_dw2-1.dll
KERNEL32.dll
msvcrt.dll
libwinpthread-1.dll
USER32.dll
libstdc++-6.dll
```

Il utilise également plusieurs fonctions communes à `Env.exe` :

```text
GetCurrentProcess
GetCurrentProcessId
GetCurrentThreadId
GetProcAddress
LoadLibraryA
TerminateProcess
VirtualProtect
VirtualQuery
```

La présence de `libwinpthread-1.dll` et de fonctions relatives aux threads indique également que l'application peut effectuer des traitements multithreadés.

---

## 5. Analyse des chaînes de caractères

Les chaînes ont été extraites avec :

```bash
strings -a -n 5 Env.exe > env_strings.txt
strings -a -el Env.exe > env_unicode.txt
```

et :

```bash
strings -a -n 5 Res.exe > res_strings.txt
strings -a -el Res.exe > res_unicode.txt
```

### Chaînes intéressantes dans Env.exe

Plusieurs chaînes relatives à la couche réseau Qt sont présentes :

```text
QAbstractSocket::SocketState
QAbstractSocket::SocketError
QSslSocket::waitForEncrypted
QSslSocket::startClientEncryption
QSslSocket::connectToHostEncrypted
```

Cela confirme que `Env.exe` est fortement lié aux communications réseau chiffrées.

Aucune URL, adresse IP ou nom de domaine en clair n'a été retrouvé avec les recherches effectuées.

---

## 6. Chaînes intéressantes dans Res.exe

L'analyse de `Res.exe` révèle plusieurs commandes particulièrement importantes :

```text
XCOPY libgcc_s_dw2-1.dll c:\WindSyst /S
XCOPY libstdc++-6.dll c:\WindSyst /S
XCOPY libwinpthread-1.dll c:\WindSyst /S
XCOPY Qt5Cored.dll c:\WindSyst /S
XCOPY Res.exe c:\WindSyst /S
XCOPY Env.exe c:\WindSyst /S
XCOPY Qt5Widgets.dll c:\WindSyst /S
XCOPY Qt5Network.dll c:\WindSyst /S
XCOPY Qt5Gui.dll c:\WindSyst /S
XCOPY Qt5Core.dll c:\WindSyst /S
```

Plusieurs plugins Qt sont également copiés dans :

```text
c:\WindSyst\platforms
```

Ces chaînes indiquent que `Res.exe` semble avoir pour rôle de **copier les différents composants du malware dans le répertoire `C:\WindSyst`**.

---

## 7. Mécanisme de persistance

Une chaîne particulièrement importante a été trouvée dans `Res.exe` :

```text
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
```

Cette clé de registre est couramment utilisée pour lancer automatiquement un programme lors de l'ouverture de session Windows.

Les chemins suivants sont également présents :

```text
C:\WindSyst\Res.exe
C:\WindSyst\Env.exe
```

La combinaison de ces éléments suggère fortement que `Res.exe` met en place un **mécanisme de persistance au démarrage de la session utilisateur**, probablement en enregistrant l'un des exécutables dans cette clé `Run`.

---

## 8. Fonctions principales identifiées

L'analyse statique permet d'identifier plusieurs comportements probables.

### Res.exe

`Res.exe` semble principalement servir à :

* copier les composants du malware dans `C:\WindSyst` ;
* copier les DLL nécessaires à son fonctionnement ;
* installer les plugins Qt nécessaires ;
* mettre en place une persistance via le registre Windows.

Il semble donc jouer le rôle de **programme d'installation / persistance**.

### Env.exe

`Env.exe` semble principalement servir à :

* lancer une interface graphique basée sur Qt ;
* établir des connexions réseau ;
* utiliser des communications chiffrées SSL/TLS ;
* charger dynamiquement des bibliothèques et fonctions.

Il semble donc jouer le rôle de **composant principal du malware**, notamment pour les communications réseau.

L'analyse statique seule ne permet cependant pas de déterminer l'adresse du serveur distant ni le contenu exact des données échangées.

---

## 9. Indicateurs de compromission

Les IoC suivants ont été extraits :

| Type         | Valeur                                                             |
| ------------ | ------------------------------------------------------------------ |
| SHA-256      | `e09ec2098363a129de143fdaf73ad6e2e61266fba3f638a25214af3a8bc8f2f2` |
| Fichier      | `Env.exe`                                                          |
| Fichier      | `Res.exe`                                                          |
| Répertoire   | `C:\WindSyst`                                                      |
| Exécutable   | `C:\WindSyst\Env.exe`                                              |
| Exécutable   | `C:\WindSyst\Res.exe`                                              |
| Clé registre | `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run`  |

Aucune adresse IP, URL ou domaine en clair n'a été identifié dans les chaînes analysées.

---

## Conclusion

L'analyse statique montre que le malware est constitué principalement de deux exécutables Windows 32 bits développés en C++ avec Qt.

`Res.exe` semble assurer l'installation des composants du malware dans le répertoire :

```text
C:\WindSyst
```

Il contient également une référence à la clé de registre :

```text
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
```

ce qui indique un mécanisme probable de persistance.

`Env.exe` utilise `Qt5Network.dll` et plusieurs fonctions de `QSslSocket`, ce qui montre qu'il dispose de capacités de communication réseau chiffrée SSL/TLS.

Les principaux comportements identifiés statiquement sont donc :

* installation dans `C:\WindSyst` ;
* persistance via le registre Windows ;
* communication réseau chiffrée ;
* chargement dynamique de fonctions et bibliothèques.
