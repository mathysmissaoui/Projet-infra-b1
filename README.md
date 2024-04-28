# Document technique projet Infra

**mathys bjorn lionel nathan**

Nous avons crée cette documentation technique pour permettre a un nouvel utilisateur du VPN de connaitre les differentes etapes de construction et d'installation pour qu'ilpuisse utilisé le VPN.



## Sommaire

- [I. Setup nouvel utilisateur](#i-setup)
  - [1. Cotés client (nouvel employé)](#2-fichier)
  - [2. Cotés serveur](#2-fichier)
- [II. Setup le Serveur vpn ](#ii-casser)
- [III. Setup le client VPN](#ii-casser)
- [IV. Conclusion ](#ii-casser)

 
# I. Setup nouvel utilisateur

 ## Cotés client (nouvel employé)

Coté client il faut ajouter la clé publique générés et l’inscrire dans le fichier de conf :  /etc/wireguard/wg0.conf

```
Sudo cat /etc/wireguard/wg0.conf
```
## Cotés serveur  

Coté serveur il faut ajouter la clé publique générés et l’inscrire dans le fichier de conf disponible en interface graphique sur WireGuard 

Avec la commande suivante vous pouvez vérifier si l’utilisateur est bien enregistré dans le VPN :

```
$ sudo wg show wgo
```

nous allons voir comment savoir si le serveur VPN tourne bien pour cela rien de plus simple il suffit d’exécuter la commande suivante : 
                        
```
Sudo systemctl status wg-quick@wg0
``` 



# II. Setup le Serveur vpn 


Tout d’abord il fallait activer le module noyau Wireguard

```
$ sudo modprobe wireguard
$ lsmod | grep wireguard

```

Ensuite il était important de charger le module wireguard de façon permanent .Cela permet charger le module du noyau wireguard de façon permanente au démarrage du système.

```
sudo echo wireguard > /etc/modules-load.d/wireguard.conf
```
   Une fois que le module noyau 'wireguard' est activé, nous avons procédé a l' installation des outils liés à Wireguard
   
```
$ sudo dnf install wireguard-tools
```


 Notre serveur Rocky Linux a maintenant le module de noyau 'wireguard' activé et les   'wireguard-tools' installés 
ensuite nous avons générer des paires de clés pour le serveur et le client Wireguard via les outils wireguard. Il nous fallait Généré une paire de clés serveur et client et Généré une paire de clés de serveur
 tout d’abord nous avous genèrer une clé privée, stocker dans un fichier dédié, dans le dossier /etc/wireguard

```
wg genkey | sudo tee /etc/wireguard/server.key
```

 ensuite il nous faut définir des permissions restrictives sur la clé.  l’autorisation par défaut en « 0400 » pour désactiver l’écriture et l’exécution à partir  d’autres personnes et groupes.

```
sudo chmod 0400 /etc/wireguard/server.key
```

on génère la clé publique, La clé publique wireguard est dérivée de la clé privée wireguard

```
sudo cat /etc/wireguard/server.key | wg pubkey | sudo tee /etc/wireguard/server.pub
```

Après avoir généré la paire de clés du serveur wireguard, nous en avons générer une nouvelle pour le client. la nouvelle paire de clés porte le nom « employee ».
 Tout d’abord on crée un dossier pour héberger les clés de nos clients

```
 sudo mkdir -p /etc/wireguard/clients
```

 on génère une clé privée, et on la stocke dans un fichier dédié, dans le dossier      /etc/wireguard

```
wg genkey | sudo tee /etc/wireguard/clients/employee
```

 il faut on définir des permissions restrictives sur la clé

 ```
sudo chmod 0400 /etc/wireguard/clients/server.key
 ```

ensuite on a génèré la clé publique

```
sudo cat /etc/wireguard/clients/server.key| wg pubkey | tee /etc/wireguard/clients/server.pub
```

il fallait crée le fichier de configuration du servreur vpn SUR LA VM 

  ```
  sudo cat /etc/wireguard/wg0.conf
[Interface]
Address = 10.107.1.0/24
SaveConfig = false
PostUp = firewall-cmd --zone=public --add-masquerade
PostUp = firewall-cmd --add-interface=wg0 --zone=public
PostDown = firewall-cmd --zone=public --remove-masquerade
PostDown = firewall-cmd --remove-interface=wg0 --zone=public
ListenPort = 13337
PrivateKey = <clé du fichier /etc/wireguard/server.key>

[Peer]
PublicKey = <clé du fichier /etc/wireguard/clients/john.pub>

  ``` 
 

Configuration de Firewalld
Nous avons configurer le pare-feu sur le serveur Wireguard.  Nous avons acheminer les trafics client vers l’interface réseau spécifique sur le serveur wireguard via firewalld..

```
ip route show default
```


Ensuite, nous avons ouvert le fichier de configuration du serveur wireguard '/etc/wireguard/wg0.conf .

```
sudo nano /etc/wireguard/wg0.conf
```
on a Ajoutez les lignes suivantes à la section '[Interface]'.

```
PostUp = firewall-cmd --zone=public --add-masquerade
PostUp = firewall-cmd --direct --add-rule ipv4 filter FORWARD 0 -i wg -o eth0 -j ACCEPT
PostUp = firewall-cmd --direct --add-rule ipv4 nat POSTROUTING 0 -o eth0 -j MASQUERADE

PostDown = firewall-cmd --zone=public --remove-masquerade
PostDown = firewall-cmd --direct --remove-rule ipv4 filter FORWARD 0 -i wg -o eth0 -j ACCEPT
PostDown = firewall-cmd --direct --remove-rule ipv4 nat POSTROUTING 0 -o eth0 -j MASQUERADE
```

Le paramètre 'PostUp' sera exécuté chaque fois que le serveur Wirguard démarrera le tunnel VPN.
Le paramètre 'PostDown' sera exécuté chaque fois que le serveur Wireguard arrêtera le tunnel VPN.

La commande 'firewall-cmd --zone=public --add-masquerade' activera la mascarade sur firewalld.

La commande 'firewall-cmd --direct --add-rule ipv4 filter FORWARD 0 -i wg -o eth0 -j ACCEPT' ajoutera firewalld rich-rule pour les trafics de l’interface wireguard à eth0.

La commande 'firewall-cmd --direct --add-rule ipv4 nat POSTROUTING 0 -o eth0 -j MASQUERADE' activera le NAT via l’interface eth0.

On a ouvert le port UDP 51420 qui sera utilisé pour les clients wirguard

```
sudo firewall-cmd --add-port=51420/udp --permanent
sudo firewall-cmd --reload

```

nous avons maintenant activé la redirection de port sur le serveur wireguard et configuré le pare-feu pour acheminer le trafic de l’interface wireguard vers l’interface réseau spécifique. Avec cela, nous avons maintenant activé la redirection de port sur le serveur wireguard et configuré le pare-feu pour acheminer le trafic de l’interface wireguard vers l’interface réseau spécifique. Vous êtes maintenant prêt à démarrer le serveur wireguard.

Démarrage du serveur Wireguard

```
sudo systemctl start wg-quick@wg0.service
sudo systemctl enable wg-quick@wg0.service
```

Exécutez le 'wg-quick up' pour démarrer le serveur wireguard et le 'wg-quick down' pour arrêter le serveur wireguard.
```
sudo wg-quick up /etc/wireguard/wg0.conf
sudo wg-quick down /etc/wireguard/wg0.conf
```

À ce stade, le serveur wireguard est en cours d’exécution sur le serveur Rocky Linux. À l’étape suivante, vous allez configurer une machine cliente et la connecter au serveur wireguard.

# III. Setup le client VPN

Au cours de cette étape, nous allons configurer un Wireguard sur un ordinateur client Linux, puis connecter l'ordinateur client au serveur Wireguard. 

Connectez-vous à la machine client et exécutez la commande dnf ci-dessous pour installer le package wireguard-tools sur votre client. Cette opération installe également la dépendance 'systemd-resolved' qui sera utilisée par wireguard-tools pour gérer le résolveur DNS.

```
sudo dnf install wireguard-tools
```

Pendant les ' wireguard-tools ', le package résolu par systemd est également installé en tant que dépendance.

Exécutez la commande ci-dessous pour démarrer et activer le ' systemd-resolved ».

```
sudo systemctl start systemd-resolved
sudo systemctl enable systemd-resolved
```
Une fois les outils wireguard installés et le systemd-resolved en cours d'exécution, vous allez ensuite configurer NetworkManager pour utiliser le ' systemd-resolved ' comme backend DNS.

Ouvrez le fichier de configuration NetworkManager ' /etc/NetworkManager/NetworkManager.conf ' à l'aide de la commande de l'éditeur nano ci-dessous.
```
sudo nano /etc/NetworkManager/NetworkManager.conf
```
nsuite, exécutez la commande ci-dessous pour supprimer le fichier '/etc/resolv.conf ' et créez un nouveau fichier de lien symbolique du 'resolv.conf ' géré par systemd-resolved.

```

rm -f /etc/resolv.con
sudo ln -s /usr/lib/systemd/resolv.conf /etc/resolv.conf
```

Redémarrez maintenant le service NetworkManager pour appliquer les modifications.
```
sudo systemctl restart NetworkManager
```
Maintenant que NetworkManager est configuré, vous êtes maintenant prêt à configurer le client Wireguard.

Créez un nouveau fichier ' /etc/wireguard/wg-client1.conf' à l'aide de l'éditeur nano ci-dessous.
```
sudo nano /etc/wireguard/wg-client1.conf
```

Ajoutez les lignes suivantes au fichier.
```
[Interface]
# Define the IP address for the client - must be matched with wg0 on Wireguard Server
Address = 10.8.0.8/24

# Private key for the client - client1.key
PrivateKey = 4FsCdtKr9GrLiX7zpNEYeqodMa5oSeHwH/m9hsNNfEs=

# Run resolvectl command
PostUp = resolvectl dns %i 1.1.1.1 9.9.9.9; resolvectl domain %i ~.
PreDown = resolvectl revert %i

[Peer]
# Public key of the Wireguard server - server.pub
PublicKey = aK+MQ48PVopb8j1Vjs8Fvgz5ZIG2k6pmVZs8iVsgr1E=

# Allow all traffic to be routed via Wireguard VPN
AllowedIPs = 0.0.0.0/0

# Public IP address of the Wireguard Server
Endpoint = 192.168.5.59:51820

# Sending Keepalive every 25 sec
PersistentKeepalive = 25
```

# III.  Conclusion

Dans ce guide, vous avez installé et configuré le serveur VPN Wireguard sur Rocky Linux . Vous avez également appris comment configurer la machine client Rocky Linux pour se connecter au serveur Wireguard. Parallèlement à cela, vous avez également appris comment activer la redirection de port et configurer NAT via pare-feu.

Vous avez surtout apris comment utiliser le Vpn dans le cadre de l'entreprise dans lequel on l'a réalisé.