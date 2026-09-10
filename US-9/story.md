# US 9 : Identification de la clé USB utilisée

L'objectif est d'identifier de manière unique la clé USB utilisée sur le poste compromis.

## Commande utilisée

```bash
lsblk -o NAME,MODEL,SERIAL,VENDOR,TRAN
```

## Résultat

```text
NAME        MODEL                     SERIAL           VENDOR   TRAN
sda         Cruzer Micro              3514931B5ED0D062 SanDisk  usb
└─sda1
```

La commande permet d'identifier le périphérique USB ainsi que son numéro de série.

| Information     | Résultat           |
| --------------- | ------------------ |
| Périphérique    | `sda`              |
| Modèle          | `Cruzer Micro`     |
| Fabricant       | `SanDisk`          |
| Interface       | `USB`              |
| Numéro de série | `3514931B5ED0D062` |

## Conclusion

La clé USB utilisée est une **SanDisk Cruzer Micro**.

Son numéro de série unique est :

```text
3514931B5ED0D062
```

