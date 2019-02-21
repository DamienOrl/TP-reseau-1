# I. Exploration du réseau d'une machine CentOS
`net1` est en /24 et possède donc **254** adresses disponibles.
`net2` est en /30 et possède donc **2** adresses disponibles : utile pour restreindre le réseau et le sécuriser.

## Paramétrage:

### Mise en place des IP statiques
Dans `/etc/sysconfig/network-scripts/ifcfg-enp0s8`:
```
DEVICE=enp0s8
NAME=enp0s8

TYPE=Ethernet
BOOTPROTO=static
IPADDR=10.1.1.2
NETMASK=255.255.255.0

ONBOOT=yes
```
Et dans `/etc/sysconfig/network-scripts/ifcfg-enp0s9`:
```
NAME=enp0s9
DEVICE=enp0s9

TYPE=Ethernet
BOOTPROTO=static
IPADDR=10.1.2.2
NETMASK=255.255.255.252

ONBOOT=yes
```

### Définir le nom de domaine:
`sudo hostname client1.tp1.b2` (à chaud, jusqu'au prochain redémarrage)
`sudo vi /etc/hostname`: `client1.tp1.b2` (à froid, de manière permanente)

301 car cURL renvoie le contenu HTML et le contenu a été déplacé vers une autre adresse.

## Opérations de base

### Sur la (ip) route, toute la sainte journée...
```
[dams@client1 ~]$ ip route show
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
10.1.1.0/24 dev enp0s8 proto kernel scope link src 10.1.1.2 metric 101
10.1.2.0/30 dev enp0s9 proto kernel scope link src 10.1.2.2 metric 102
```
Composants de la ligne:
+ Adresse réseau/masque  
+ Device: nom de l'interface réseau
+ *vocabulaire spécifique linux*
+ Adresse IP normalement présente sur cette machine
+ Priorité de la route (Toutes différentes ici alors BALEC!)

`sudo ip route del 10.1.2.0/30` : on enlève la route de la deuxième carte réseau, on tente de `ping 10.1.2.2` : __FAIL__ (ou WIN dans ce cas)


Après avoir remis la route dans `/etc/sysconfig/network-scripts/route-enp0s9` et un reboot, 2ème essai: __Tout fonctionne!__

### Nos chers voisins:

```
[dams@client1 ~]$ ip neigh show
10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:1b REACHABLE
```
+ IP voisin
+ Interface réseau
+ Link Layer ADDRess MAC (adresse physique du voisin)
+ Statut: Atteignable (REACHABLE) ou liaison périmée (STALE)

Après un `ping 10.1.2.1`:
```
[dams@client1 ~]$ ip neigh show
10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:1b REACHABLE
10.1.2.1 dev enp0s9 lladdr 0a:00:27:00:00:04 REACHABLE
```
L' IP est enregistrée dans la table ARP pour faciliter les communications dans un futur proche.

# II. Communication simple entre deux machines

## Sortons les chatons!
Via `netcat`, nous allons établir une liaison entre les machines en utilisant deux protocoles différents:
### Le protocole UDP:
```
[dams@client1 ~]$ nc -u -l 8888
Allow?
Oui!
Coucou!

[dams@client2 ~]$ nc -u 10.1.2.2 8888
Allow?
Oui!
Coucou!

[dams@client1 ~]$ ss -unp
Recv-Q       Send-Q       Local Address:Port     Peer Address:Port
0            0            10.1.2.2:8888          10.1.2.1:57601
users:(("nc",pid=3091,fd=4))

[dams@client2 ~]$ ss -unp
Recv-Q       Send-Q       Local Address:Port     Peer Address:Port
0            0            10.0.2.15:49265        10.1.2.2:8888
users:(("nc",pid=2969,fd=3))
```

### Et le protocole TCP:
```
[dams@client1 ~]$ nc -l 8888
Et maintenant?
C'est bon!

[dams@client2 ~]$ nc 10.1.2.2 8888
Et maintenant?
C'est bon!

[dams@client1 ~]$ ss -tnp
State    Recv-Q   Send-Q  Local Address:Port              Peer Address:Port         
ESTAB    0        0       10.1.2.2:22                     10.1.2.1:65494
ESTAB    0        0       10.1.2.2:8888                   10.1.2.1:65497               users:(("nc",pid=3147,fd=5))
ESTAB    0        0       10.1.2.2:22                     10.1.2.1:65233

[dams@client2 ~]$ ss -tnp
State    Recv-Q Send-Q    Local Address:Port              Peer Address:Port
ESTAB    0      0         10.0.2.15:45984                 10.1.2.2:8888                users:(("nc",pid=3027,fd=3))
ESTAB    0      0         10.1.1.3:22                     10.1.1.1:65279
ESTAB    0      0         10.1.1.3:22                     10.1.1.1:65491
```

# III. Routage statique simple

## Activer l'IPv4 forwarding sur `client1`:
```
[dams@client1 ~]$ sudo sysctl -w net.ipv4.ip_forward=1
[sudo] password for dams:
net.ipv4.ip_forward = 1
```
## Ajouter une route vers `net2` sur `client2`:
```
[dams@client2 ~]$ sudo vi /etc/sysconfig/network-scripts/route-enp0s9
10.1.2.1/30 dev enp0s9 proto kernel scope link src 10.1.2.2 metric 101
```

Est-ce que ça fonctionne?
```
[dams@client2 network-scripts]$ ping -c 4 10.1.2.1
PING 10.1.2.1 (10.1.2.1) 56(84) bytes of data.
64 bytes from 10.1.2.1: icmp_seq=1 ttl=127 time=1.14 ms
64 bytes from 10.1.2.1: icmp_seq=2 ttl=127 time=0.880 ms
64 bytes from 10.1.2.1: icmp_seq=3 ttl=127 time=0.667 ms
64 bytes from 10.1.2.1: icmp_seq=4 ttl=127 time=1.13 ms

--- 10.1.2.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 0.667/0.956/1.146/0.197 ms
```
***Oui!*** Maintenant, voyons voir par où nos paquets sont passés...
```
[dams@client2 network-scripts]$ traceroute 10.1.2.1
traceroute to 10.1.2.1 (10.1.2.1), 30 hops max, 60 byte packets
 1  gateway (10.0.2.2)  0.476 ms  0.485 ms  0.449 ms
 2  gateway (10.0.2.2)  1.170 ms  1.444 ms  1.073 ms
```
