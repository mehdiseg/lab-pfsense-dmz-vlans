# Commandes VBoxManage réellement utilisées

Reproduites telles quelles (Windows, VirtualBox 7.2.6). Adapter les chemins sur une autre machine.

## Créer la VM et ses réseaux

```bash
VBoxManage createvm --name pfsense-lab --ostype "FreeBSD_64" --register
VBoxManage modifyvm pfsense-lab --memory 2048 --cpus 1 --firmware bios --audio-driver none --boot1 dvd --boot2 disk --boot3 none --boot4 none
VBoxManage modifyvm pfsense-lab --nic1 nat --nictype1 82540EM
VBoxManage modifyvm pfsense-lab --nic2 hostonly --nictype2 82540EM --hostonlyadapter2 "VirtualBox Host-Only Ethernet Adapter"
VBoxManage modifyvm pfsense-lab --nic3 intnet --nictype3 82540EM --intnet3 "lab-dmz"
VBoxManage createmedium disk --filename pfsense-lab.vdi --size 8192 --format VDI
VBoxManage storagectl pfsense-lab --name SATA --add sata --controller IntelAhci
VBoxManage storagectl pfsense-lab --name IDE --add ide
VBoxManage storageattach pfsense-lab --storagectl SATA --port 0 --device 0 --type hdd --medium pfsense-lab.vdi
VBoxManage storageattach pfsense-lab --storagectl IDE --port 0 --device 0 --type dvddrive --medium pfsense.iso
```

## Démarrer sans interface graphique, et piloter le clavier à l'aveugle

```bash
VBoxManage startvm pfsense-lab --type headless
VBoxManage controlvm pfsense-lab screenshotpng capture.png   # lire l'écran
VBoxManage controlvm pfsense-lab keyboardputscancode 1c 9c   # Entrée (appui puis relâchement)
VBoxManage controlvm pfsense-lab keyboardputstring "192.168.56.2"   # taper du texte
```

## Après l'installation : éjecter le CD avant le redémarrage

Sinon le BIOS rebootera sur l'installateur au lieu du disque installé (le contrôleur IDE ne permet pas de détacher le lecteur pendant que la VM tourne, mais on peut éjecter le disque qu'il contient) :

```bash
VBoxManage storageattach pfsense-lab --storagectl IDE --port 0 --device 0 --type dvddrive --medium emptydrive
```

## Blocage rencontré : écran figé après le premier redémarrage

Après l'installation, le premier redémarrage ACPI a laissé l'écran figé sur « Loading kernel... » sans plus aucune progression (CPU pourtant actif). Un `poweroff` suivi d'un nouveau `startvm` (démarrage à froid, pas un redémarrage) a résolu le problème du premier coup :

```bash
VBoxManage controlvm pfsense-lab poweroff
VBoxManage startvm pfsense-lab --type headless
```

## Configurer via l'interface web sans navigateur piloté à l'aveugle

Le webConfigurator de pfSense protège chaque formulaire par un jeton anti-CSRF lié à la session. La méthode la plus fiable en ligne de commande :

```bash
# 1. Récupérer la page et le jeton, avec un pot de cookies
curl -s -c jar.txt "http://192.168.56.2/interfaces.php?if=opt1" -o page.html
CSRF=$(grep -oE "name='__csrf_magic' value=\"[^\"]+\"" page.html | sed -E 's/.*value="([^"]+)"/\1/')

# 2. Soumettre le même formulaire avec les mêmes champs, avec le même pot de cookies
curl -s -b jar.txt -c jar.txt \
  --data-urlencode "__csrf_magic=$CSRF" \
  --data-urlencode "if=opt1" \
  --data-urlencode "enable=yes" \
  --data-urlencode "type=staticv4" \
  --data-urlencode "ipaddr=172.16.0.1" \
  --data-urlencode "subnet=24" \
  --data-urlencode "gateway=none" \
  --data-urlencode "save=Save" \
  "http://192.168.56.2/interfaces.php?if=opt1"
```

Piège rencontré : sur une interface de type LAN/DMZ, le champ `gateway` doit être envoyé avec la valeur `none` — l'omettre déclenche l'erreur « The field Gateway is required », alors que rien dans l'interface ne le rend visuellement obligatoire.
