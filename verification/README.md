# Vérification réelle — 22 septembre 2026

Contrairement aux autres labs « à réaliser », celui-ci a été **entièrement automatisé et exécuté pour de vrai**, pas seulement vérifié commande par commande : une vraie machine virtuelle VirtualBox, un vrai pfSense CE 2.7.2 installé sans aucune interaction manuelle (clavier piloté par scancodes PS/2 via `VBoxManage`, écran lu par captures d'écran réelles), puis configuré via son interface web réelle.

## Ce qui a été fait, réellement

1. **VM VirtualBox** créée avec trois cartes réseau : NAT (WAN), réseau « hostonly » (LAN, pour que la machine hôte puisse atteindre l'interface web), réseau interne `lab-dmz` (DMZ) — voir `creer-vm.md` pour les commandes exactes.
2. **Téléchargement et vérification** de l'ISO officielle `pfSense-CE-2.7.2-RELEASE-amd64.iso` (SHA-256 vérifié face à la somme publiée par le mirroir).
3. **Installation entièrement automatisée** : licence acceptée, partitionnement UFS sur disque entier (MBR), extraction, redémarrage — le tout piloté à l'aveugle puis vérifié à chaque étape par une vraie capture d'écran (`VBoxManage controlvm ... screenshotpng`), sans jamais inventer un résultat.
4. **Premier démarrage** : un blocage matériel est survenu une fois (écran figé après un redémarrage ACPI) — corrigé par un arrêt forcé (`poweroff`) suivi d'un démarrage propre, qui a fonctionné du premier coup.
5. **Assignation des interfaces** via la console (WAN=em0, LAN=em1, DMZ/OPT1=em2), IP du LAN réglée sur `192.168.56.2/24` pour être joignable depuis l'hôte (le README utilise `192.168.1.1` : ce choix ne change que l'adresse, pas la logique).
6. **Connexion réelle** à l'interface web (`capture-login.png`), confirmée par le propre journal système de pfSense : `php-fpm[...]: /index.php: Successful login for user 'admin' from: 192.168.56.1`.
7. **Interface DMZ configurée** : activée, IPv4 statique `172.16.0.1/24`, sans passerelle (comme pour un LAN).
8. **Règle de pare-feu** « Bloque DMZ vers LAN » (destination : LAN subnets) créée **avant** la règle « Autorise DMZ vers internet » (destination : any) — l'ordre visible dans `capture-regles-dmz.png` correspond exactement à ce que demande le README.
9. **Redirection de port NAT** WAN:80 → 172.16.0.10:80, visible dans `capture-nat-portforward.png`.
10. Les deux jeux de règles ont été **appliqués** (« The changes have been applied successfully »), donc réellement chargés dans le pare-feu `pf`, pas seulement enregistrés dans la configuration.

## Comment la configuration a été appliquée

Pas en cliquant dans un navigateur piloté à l'aveugle (trop fragile en mode sans tête), mais en soumettant directement les formulaires du webConfigurator avec `curl` : lecture de la page pour extraire le jeton anti-CSRF (`__csrf_magic`, lié à la session), puis `POST` des mêmes champs que le formulaire HTML réel, avec le même fichier de session que l'interface graphique. Les captures d'écran ci-dessus sont prises **après coup**, avec un vrai navigateur (Edge sans tête) authentifié avec le cookie de session obtenu, pour prouver que le résultat est bien visible dans l'interface elle-même — pas seulement dans une réponse HTTP.

## Ce qui n'a pas été vérifié

- **Aucun serveur web n'écoute réellement à `172.16.0.10`** : la redirection de port est configurée et appliquée, mais rien ne répond de l'autre côté (aucune deuxième machine n'a été installée sur le réseau interne `lab-dmz`, isolé et non accessible depuis l'hôte). Le test complet « `curl http://<IP_WAN>/` renvoie la page du serveur » décrit dans le README reste à faire.
- **Mot de passe administrateur laissé par défaut** (`pfsense`) : pfSense l'affiche lui-même en avertissement rouge sur chaque page. À changer avant tout usage réel.
- **Les tests de connectivité **depuis** un poste LAN ou DMZ** (`ping`, blocage réellement constaté) n'ont pas été faits : je n'ai pas mis de troisième machine sur ces réseaux internes.

## Fichiers de ce dossier

| Fichier | Contenu |
|---|---|
| `capture-login.png` | vraie page de connexion pfSense, servie par la VM |
| `capture-status-interfaces.png` | Status → Interfaces : WAN, LAN et DMZ, avec de vraies adresses et compteurs de paquets |
| `capture-regles-dmz.png` | Firewall → Rules → DMZ : les deux règles, dans le bon ordre |
| `capture-nat-portforward.png` | Firewall → NAT → Port Forward : la redirection WAN:80 → 172.16.0.10:80 |
