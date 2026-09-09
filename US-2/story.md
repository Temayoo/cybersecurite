# US 2 : Vérification de la reconnaissance

Commande exécutée pour obtenir le hash :

```bash
sha256sum Env.exe
```

Résultat obtenu :

```text
e09ec2098363a129de143fdaf73ad6e2e61266fba3f638a25214af3a8bc8f2f2  Env.exe
```

Utilisation de VirusTotal afin de vérifier si le malware est déjà connu.

Le fichier est déjà référencé et détecté comme malveillant par **26 moteurs antivirus sur 71**.

| Information     | Résultat                  |
| --------------- | ------------------------- |
| Statut          | Déjà référencé            |
| Threat category | `trojan`                  |
| Family labels   | `keylogger`, `malgent`    |
| Tag             | `check-user-input`        |
| Creation Time   | 22/12/2017 à 00:40:43 UTC |
| Last Submission | 07/09/2026 à 14:00:04 UTC |

## Conclusion

L'exécutable `Env.exe` est **déjà connu et référencé** dans VirusTotal.

Son empreinte SHA-256 correspond à un échantillon déjà analysé et détecté comme malveillant par plusieurs moteurs antivirus.

Les informations disponibles l'associent notamment à un **Trojan / Keylogger**.

Il ne s'agit donc pas, d'après la base consultée, d'une menace inconnue.
