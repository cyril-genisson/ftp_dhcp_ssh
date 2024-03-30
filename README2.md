# **Réseau Virtuel**
## Objectif : 
Configurer un envirronement réseau virtuel à l'aide de deuc machines virtuelles debian,
en mettant en place un serveur DHCP et DNS sur la premiére machine, ainsi q'un serveur client FTP
avec SSH sur la seconde machine 

## Voici les tâches à accomplir : 
- Installation de Debian sans interface graphique
- Mise à jour des systèmes
- Configuration du serveur DHCP
- Installation du Serveur FTP et SSH
- Installation du serveur DNS
- Test de connexion au serveur FTP
- Paramètres de Sécurité Additionnels

## Création des VM
Tout d'abord, on configure le réseau dans lesquels nos machines vont pouvoir communiquer,nous utiliserons une adresse IP de type B : 
Adresse IPv4 du réseau : 172.18.0.0/16
Et on désactive le DHCP : 
(Screen/VM.png "Config Réseau")

## Mise à jour 
Commande : 
````shell
apt update
apt upgrade
````
## Configuration de DHCP :
Installation des paquets sur la première machine  : 
````shell
apt install openssh-server isc-dhcp-server bind9 bin9-utils
````
On commence ensuite par modifier l'interface réseau pour donner une adresse static a note host dans 
" /etc/network/interfaces" : 
````shell
auto lo
iface lo inet loopback

allow hotplug enp0s3
  address 172.18.0.3/16
  dns-nameservers 172.18.0.3
  dns-domain ftp.com
  gateway 172.18.0.2
````
![interface](https://github.com/cyril-genisson/ftp_dhcp_ssh/assets/147488564/343d6776-6e83-4b85-990e-99be7d35f1d4)

On configure ensuite notre serveur pour allouer les adresse IP de connecter, je vais aussi dire à DHCP de donner a ma deuxieme machine une adresse fixe, pour cela il faut éditer le fichier "/etc/dhcp/dhcpd.conf" et 
l'interface dans le fichier "/etc/default/isc-dhcp-server: 
````shell
nano /etc/default/isc-dhcp-server
INTERFACESv4="enp0s3"

nano /etc/dhcp/dhcpd.conf
authoritative; 
subnet 172.18.0.0 netmask 255.255.0.0 {
  range 172.18.0.10 172.18.0.20 ;
  option domain-name "ftp.com";
  option domain-name-server ns;
  option broadcast-address 172.18.255.255;
  option routers 172.18.0.1;
  default-lease-time 5200 ;
  max-lease-time 6200;
}

host ftp {
  hardware ethernet 08:00:27:81:f6:ce ; # Adresse MAC de ma deuxième machine
  fixed-address 172.18.0.4; # adresse IP de mon DNS 
}
````
![interfacepv4](https://github.com/cyril-genisson/ftp_dhcp_ssh/assets/147488564/d6b01357-6324-40c3-88c8-ddc6a65b8c82)

![dhcpconf](https://github.com/cyril-genisson/ftp_dhcp_ssh/assets/147488564/457db805-e487-46c3-b9fb-4109d2e70276)

C'est la que je me rend compte que j'aurais du créer mon serveur DNS car l'IP static pour ma 2ème machine n'a pas était modifier celon mes choix 

## Configuration du DNS 
Installation de bind9 sur la première machine : 
````shell
apt install bind9 bind9-utils
````

Enregistrement du fichier de zone dans le fichier "/etc/bind/named.conf.local/" : 
````shell
nano /etc/bind/named.conf.local

zone "ftp.com" {
  type master;
  file "/etc/bind/db.ftp.com";
};
````
![DNSnamedconf](https://github.com/cyril-genisson/ftp_dhcp_ssh/assets/147488564/349ceaab-4b92-4b4e-bad6-f123a483b60b)

Maintenant on passe à la création de notre fichier de zone pour notre DNS : 
````shell
;
; BIND data file for ftp.com domain
;
$TTL    86400
@       IN      SOA     ftp.com. admin.ftp.com. (
                     2024033001         ; Serial
                          86400         ; Refresh
                           3600         ; Retry
                           7200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns
@       IN      A       172.18.0.3
ns      IN      A       172.18.0.3
dns     IN      A       172.18.0.4 #adresse ip de notre deuxème machine

````

![dbftp](https://github.com/cyril-genisson/ftp_dhcp_ssh/assets/147488564/620b0ffd-6bc4-4180-b8f8-dd6c96868400)





