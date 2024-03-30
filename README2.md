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

