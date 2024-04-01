# Réseau virtuel
## Objectif:
Configurer un environnement réseau virtuel à l'aide de
deux machines virtuelles Debian, en mettant en place deux
services DHCP / DNS sur la première machine, ainsi qu'un serveur
FTP avec SSH sur la seconde machine.

Liste des tâches à accomplir:
- Installation de Debian sans interface graphique
- Mise à jour des systèmes
- Configuration du serveur DHCP
- Installation du serveur FTP et SSH
- Installation du serveur DNS
- Test de connexion au serveur SFTP
- Paramètres de sécurité additionnels

 
__Remettons un peu d'ordre dans le propos__:
l'idée principale est d'avoir deux serveurs:
- ns.ftp.com: ipv4 172.16.0.3/16 - Services: ssh, dhcp, dns
- dns.ftp.com: ipv4 172.16.0.4/16 - Services: ftp, ssh / sftp 

connectés à une passerelle (gw.ftp.com 172.16.0.2/16)

````mermaid
graph LR
        A[ns] <--> C[gw / router]
        B[dns] <--> C
        C <--> D{INTERNET}
````
## Configuration de _ns.ftp.com_:
### Installation des paquets et configuration de l'interface ethernet
````shell
# Installation des paquets pour la machine ns
apt install vim openssh-server isc-dhcp-server bind9 bind9-utils man bash-completion 2>&1 /dev/null &
cat > /etc/network/interfaces << EOF

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens33
iface ens33 inet static
  address 172.16.0.3/16
  dns-nameservers 172.16.0.3
  dns-domain ftp.com
  gateway 172.16.0.2
EOF
````

### Configuration du serveur DNS (paquet bind9)
````shell
# On change les options pour sécuriser le serveur /etc/bind/named.conf.options
mv /etc/bind/named.conf.options /etc/bind/named.conf.options.save
cat > /etc/bind/named.conf.options << EOF
options {
  directory "/var/cache/bind";
  allow-query { 172.16.0.0/16; };
  allow-transfer { none; };
  allow-recursion { 172.16.0.0/16; };
  forwarders { 8.8.8.8; };
  dnssec-validation auto;
  listen-on-v6 { any; };
};

EOF

mv /etc/bind/named.conf.local /etc/bind/named.conf.local.save
cat > /etc/bind/named.conf.local << EOF
zone "ftp.com" {
  type master;
  file "/etc/bind/db.ftp.com";
};

zone "16.172.in-addr.arpa" {
  type master
  file "/etc/bind/db.16.172.in-addr.arpa";
};

EOF

cat > /etc/bind/db.ftp.com << EOF
;
; BIND data file for ftp.com domain
;
$TTL    86400
@       IN      SOA     ftp.com. admin.ftp.com. (
                     2024032501         ; Serial
                          86400         ; Refresh
                           3600         ; Retry
                           7200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns
@       IN      A       172.16.0.3
gw      IN      A       172.16.0.2
ns      IN      A       172.16.0.3
dns     IN      A       172.16.0.4

EOF

cat > /etc/bind/db.16.172.in-addr.arpa << EOF
;
; BIND reverse data file for ftp.com domain
;
$TTL    86400
@       IN      SOA     ftp.com. admin.ftp.com. (
                      2024032501        ; Serial
                          86400         ; Refresh
                           3600         ; Retry
                           7200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns
2.0     IN      PTR     gw
3.0     IN      PTR     ns
4.0     IN      PTR     dns

EOF
````
__On vérifie que tout est opérationnel__
````shell
systemctl enable --now named &&
systemctl status named
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; preset: enabled)
     Active: active (running) since Fri 2024-03-29 11:54:22 CET; 10min ago
       Docs: man:named(8)
   Main PID: 2318 (named)
     Status: "running"
      Tasks: 8 (limit: 2265)
     Memory: 45.5M
        CPU: 134ms
     CGroup: /system.slice/named.service
             └─2318 /usr/sbin/named -f -u bind
````

### Configuration du serveur DHCP (paquet isc-dhcp-server)
````shell
mv /etc/default/isc-dhcp-server /etc/default/isc-dhcp-server.save
cat > /etc/default/isc-dhcp-server << EOF
INTERFACESv4="ens33"
EOF

mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.save
cat > /etc/dhcp/dhcpd.conf << EOF
authoritative;
subnet 172.16.0.0 netmask 255.255.0.0 {
    range 172.16.0.128 172.16.0.254;
    option domain-name "ftp.com";
    option domain-name-servers ns;
    option broadcast-address 172.16.255.255;
    option routers gw;
    default-lease-time 600;
    max-lease-time 7200;
}

host ftp {
    hardware ethernet 00:0c:29:24:6e:eb;
    fixed-address dns.ftp.com;
}
EOF
````

````shell
systemctl enable --now isc-dhcp-server &&
systemctl status isc-dhcp-server
● isc-dhcp-server.service - LSB: DHCP server
     Loaded: loaded (/etc/init.d/isc-dhcp-server; generated)
     Active: active (running) since Thu 2024-03-28 20:25:13 CET; 7s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 1228 ExecStart=/etc/init.d/isc-dhcp-server start (code=exited, status=0/SUCCESS)
      Tasks: 1 (limit: 2265)
     Memory: 5.1M
        CPU: 53ms
     CGroup: /system.slice/isc-dhcp-server.service
             └─1116 /usr/sbin/dhcpd -4 -q -cf /etc/dhcp/dhcpd.conf
````

## Configuration de dns.ftp.com
### Installation des paquets
````shell
apt install vim openssh-server proftpd-basic bash-completion &&
systemctl restart networking
````
### Création de l'utilisateur
````shell
useradd -m -c "La Plateforme" -s /bin/bash laplateforme
echo 'laplateforme:Marseille13!' | chpasswd
````

### Configuration du serveur SFTP
````shell
# /etc/ssh/sshd_conf
#Changement du port de connexion
PORT 6500

# Ajout du paramètre permettant la connexion qu'à l'utilisateur
# laplateforme
AllowUsers laplateforme
````
On recharge le service
````shell
systemctl restart sshd &&
systemctl status sshd
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Sun 2024-03-28 20:27:11 CEST; 7s ago
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 635 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 680 (sshd)
      Tasks: 1 (limit: 2265)
     Memory: 5.9M
        CPU: 628ms
     CGroup: /system.slice/ssh.service
             └─680 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
````

Vérification sur la machine 'dns.ftp.com':
````shell
kaman@ftp:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:24:6e:eb brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 172.16.0.4/16 brd 172.16.255.255 scope global dynamic ens33
       valid_lft 517sec preferred_lft 517sec
    inet6 fe80::20c:29ff:fe24:6eeb/64 scope link
       valid_lft forever preferred_lft forever

kaman@ftp:~$ ping -c1 gw
PING gw.ftp.com (172.16.0.2) 56(84) bytes of data.
64 bytes from 172.16.0.2 (172.16.0.2): icmp_seq=1 ttl=128 time=0.337 ms

--- gw.ftp.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.337/0.337/0.337/0.000 ms
kaman@ftp:~$ ping -c1 google.fr
PING google.fr (216.58.198.67) 56(84) bytes of data.
64 bytes from 216.58.198.67: icmp_seq=1 ttl=128 time=3.93 ms

--- google.fr ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 3.934/3.934/3.934/0.000 ms
````

Le reste de la démonstration se fait directement sur les machines virtuelles!!!
