# TP3 : On va router des trucs

Au menu de ce TP, on va revoir un peu ARP et IP histoire de **se mettre en jambes dans un environnement avec des VMs**.

Puis on mettra en place **un routage simple, pour permettre à deux LANs de communiquer**.

![Reboot the router](./pics/reboot.jpeg)

## Sommaire

- [TP3 : On va router des trucs](#tp3--on-va-router-des-trucs)
  - [Sommaire](#sommaire)
  - [0. Prérequis](#0-prérequis)
  - [I. ARP](#i-arp)
    - [1. Echange ARP](#1-echange-arp)
    - [2. Analyse de trames](#2-analyse-de-trames)
  - [II. Routage](#ii-routage)
    - [1. Mise en place du routage](#1-mise-en-place-du-routage)
    - [2. Analyse de trames](#2-analyse-de-trames-1)
    - [3. Accès internet](#3-accès-internet)
  - [III. DHCP](#iii-dhcp)
    - [1. Mise en place du serveur DHCP](#1-mise-en-place-du-serveur-dhcp)
    - [2. Analyse de trames](#2-analyse-de-trames-2)

## 0. Prérequis

➜ Pour ce TP, on va se servir de VMs Rocky Linux. 1Go RAM c'est large large. Vous pouvez redescendre la mémoire vidéo aussi.  

➜ Vous aurez besoin de deux réseaux host-only dans VirtualBox :

- un premier réseau `10.3.1.0/24`
- le second `10.3.2.0/24`
- **vous devrez désactiver le DHCP de votre hyperviseur (VirtualBox) et définir les IPs de vos VMs de façon statique**

➜ Les firewalls de vos VMs doivent **toujours** être actifs (et donc correctement configurés).

➜ **Si vous voyez le p'tit pote 🦈 c'est qu'il y a un PCAP à produire et à mettre dans votre dépôt git de rendu.**

## I. ARP

Première partie simple, on va avoir besoin de 2 VMs.

| Machine  | `10.3.1.0/24` |
|----------|---------------|
| `john`   | `10.3.1.11`   |
| `marcel` | `10.3.1.12`   |

```schema
   john               marcel
  ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     │
  └─────┘    └───┘    └─────┘
```

> Référez-vous au [mémo Réseau Rocky](../../cours/memo/rocky_network.md) pour connaître les commandes nécessaire à la réalisation de cette partie.

### 1. Echange ARP

🌞**Générer des requêtes ARP**

- effectuer un `ping` d'une machine à l'autre
```
john to marcel


[logards@localhost ~]$ ping 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=64 time=1.11 ms
64 bytes from 10.3.1.11: icmp_seq=2 ttl=64 time=1.13 ms
64 bytes from 10.3.1.11: icmp_seq=3 ttl=64 time=1.02 ms
64 bytes from 10.3.1.11: icmp_seq=4 ttl=64 time=0.750 ms
64 bytes from 10.3.1.11: icmp_seq=5 ttl=64 time=0.721 ms
64 bytes from 10.3.1.11: icmp_seq=6 ttl=64 time=1.19 ms
64 bytes from 10.3.1.11: icmp_seq=7 ttl=64 time=1.09 ms
64 bytes from 10.3.1.11: icmp_seq=8 ttl=64 time=1.17 ms
64 bytes from 10.3.1.11: icmp_seq=9 ttl=64 time=0.933 ms
^C
--- 10.3.1.11 ping statistics ---
9 packets transmitted, 9 received, 0% packet loss, time 8053ms
rtt min/avg/max/mdev = 0.721/1.013/1.191/0.165 ms


marcel to john

[logards@localhost ~]$ ping 10.3.1.12
PING 10.3.1.12 (10.3.1.12) 56(84) bytes of data.
64 bytes from 10.3.1.12: icmp_seq=1 ttl=64 time=0.786 ms
64 bytes from 10.3.1.12: icmp_seq=2 ttl=64 time=0.893 ms
64 bytes from 10.3.1.12: icmp_seq=3 ttl=64 time=1.01 ms
^C
--- 10.3.1.12 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 0.786/0.897/1.013/0.092 ms

```
- observer les tables ARP des deux machines
- repérer l'adresse MAC de `john` dans la table ARP de `marcel` et vice-versa
```
adresse mac marcel
[logards@localhost ~]$ ip neigh show 10.3.1.12
10.3.1.12 dev enp0s8 lladdr 08:00:27:7e:00:07 STALE

adresse mac john 
[logards@localhost ~]$ ip neigh show 10.3.1.11
10.3.1.11 dev enp0s8 lladdr 08:00:27:5f:d1:a5 STALE
```
- prouvez que l'info est correcte (que l'adresse MAC que vous voyez dans la table est bien celle de la machine correspondante)
  - une commande pour voir la MAC de `marcel` dans la table ARP de `john`
```
  [logards@localhost ~]$ ip neigh show 10.3.1.12
  10.3.1.12 dev enp0s8 lladdr 08:00:27:7e:00:07 STALE
```
  - et une commande pour afficher la MAC de `marcel`, depuis `marcel`
```
[logards@localhost ~]$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:7e:00:07 brd ff:ff:ff:ff:ff:ff
```

### 2. Analyse de trames

🌞**Analyse de trames**

- utilisez la commande `tcpdump` pour réaliser une capture de trame
- videz vos tables ARP, sur les deux machines, puis effectuez un `ping`
```
[logards@localhost ~]$ sudo tcpdump -i enp0s8

14:24:50.805405 IP 10.3.1.11 > localhost.localdomain: ICMP echo request, id 1, seq 6, length 64
14:24:50.805539 IP localhost.localdomain > 10.3.1.11: ICMP echo reply, id 1, seq 6, length 64
```

[arp3](./tp3_arp)


🦈 **Capture réseau `tp3_arp.pcapng`** qui contient un ARP request et un ARP reply

> **Si vous ne savez pas comment récupérer votre fichier `.pcapng`** sur votre hôte afin de l'ouvrir dans Wireshark, et me le livrer en rendu, demandez-moi.

## II. Routage

Vous aurez besoin de 3 VMs pour cette partie. **Réutilisez les deux VMs précédentes.**

| Machine  | `10.3.1.0/24` | `10.3.2.0/24` |
|----------|---------------|---------------|
| `router` | `10.3.1.254`  | `10.3.2.254`  |
| `john`   | `10.3.1.11`   | no            |
| `marcel` | no            | `10.3.2.12`   |

> Je les appelés `marcel` et `john` PASKON EN A MAR des noms nuls en réseau 🌻

```schema
   john                router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └───┘    └─────┘    └───┘    └─────┘
```

### 1. Mise en place du routage

🌞**Activer le routage sur le noeud `router`**

```
[logards@localhost ~]$ sudo firewall-cmd --list-all
[sudo] password for logards: 
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s8 enp0s9
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[logards@localhost ~]$ sudo firewall-cmd --get-active-zone
public
  interfaces: enp0s8 enp0s9
[logards@localhost ~]$ sudo firewall-cmd --add-masquerade --zone=public
success
[logards@localhost ~]$ sudo firewall-cmd --add-masquerade --zone=public --permanent
success
```

> Cette étape est nécessaire car Rocky Linux c'est pas un OS dédié au routage par défaut. Ce n'est bien évidemment une opération qui n'est pas nécessaire sur un équipement routeur dédié comme du matériel Cisco.

🌞**Ajouter les routes statiques nécessaires pour que `john` et `marcel` puissent se `ping`**

- il faut taper une commande `ip route add` pour cela, voir mémo
- il faut ajouter une seule route des deux côtés
- une fois les routes en place, vérifiez avec un `ping` que les deux machines peuvent se joindre

```
[logards@localhost ~]$ sudo firewall-cmd --list-all
[sudo] password for logards: 
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s8 enp0s9
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[logards@localhost ~]$ sudo firewall-cmd --get-active-zone
public
  interfaces: enp0s8 enp0s9
[logards@localhost ~]$ sudo firewall-cmd --add-masquerade --zone=public
success
[logards@localhost ~]$ sudo firewall-cmd --add-masquerade --zone=public --permanent
success
```

![THE SIZE](./pics/thesize.png)
### 2. Analyse de trames

🌞**Analyse des échanges ARP**

- videz les tables ARP des trois noeuds
- effectuez un `ping` de `john` vers `marcel`
- regardez les tables ARP des trois noeuds
```
router
[logards@localhost ~]$ ip n s
10.3.1.0 dev enp0s8 lladdr 0a:00:27:00:00:01 DELAY
10.3.1.11 dev enp0s8 lladdr 08:00:27:5f:d1:a5 STALE
10.3.2.12 dev enp0s9 lladdr 08:00:27:7e:00:07 STALE

john
[logards@localhost ~]$ ip n s
10.3.1.254 dev enp0s8 lladdr 08:00:27:7a:89:aa STALE
10.3.1.0 dev enp0s8 lladdr 0a:00:27:00:00:01 REACHABLE

marcel

[logards@localhost ~]$ ip n s
10.3.2.254 dev enp0s8 lladdr 08:00:27:ae:6f:1b STALE
10.3.2.0 dev enp0s8 lladdr 0a:00:27:00:00:02 DELAY
```
- essayez de déduire un peu les échanges ARP qui ont eu lieu
- répétez l'opération précédente (vider les tables, puis `ping`), en lançant `tcpdump` sur `marcel`
- **écrivez, dans l'ordre, les échanges ARP qui ont eu lieu, puis le ping et le pong, je veux TOUTES les trames** utiles pour l'échange

Par exemple (copiez-collez ce tableau ce sera le plus simple) :

| ordre | type trame  | IP source | MAC source              | IP destination | MAC destination            |
|-------|-------------|-----------|-------------------------|----------------|----------------------------|
| 1     | Requête ARP | PcsCompu_7e:00:07          | PcsCom | Broadcast              | Broadcast `FF:FF:FF:FF:FF` |
| 2     | Réponse ARP | PcsCompu_ae:6f:1b         | 08:00:27:ae:6f:1b                       | PcsCompu_7e:00:07              | 08:00:27:7E:00:07    |
| ...   | ...         | ...       | ...                     |                |                            |
| 3    | Ping        | ICMP        | 10.3.2.12                       | 08:00:27:7e:00:07              |   10.3.1.11                        |08:00:27:5f:d1:a5
| 4     | Pong        | ICMP         | 10.3.1.11                       |    08:00:27:5f:d1:a5         |      10.3.2.12                   |08:00:27:7e:00:07

> Vous pourriez, par curiosité, lancer la capture sur `john` aussi, pour voir l'échange qu'il a effectué de son côté.

[tp3_routage_marcel](./tp3_routage_marcel.pcap)

🦈 **Capture réseau `tp3_routage_marcel.pcapng`**

### 3. Accès internet

🌞**Donnez un accès internet à vos machines**

- ajoutez une carte NAT en 3ème inteface sur le `router` pour qu'il ait un accès internet
- ajoutez une route par défaut à `john` et `marcel`
  - vérifiez que vous avez accès internet avec un `ping`
  - le `ping` doit être vers une IP, PAS un nom de domaine
```
  [logards@localhost ~]$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=61 time=28.6 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=61 time=27.3 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=61 time=24.8 ms
^C
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 24.832/26.898/28.555/1.547 ms
```

- donnez leur aussi l'adresse d'un serveur DNS qu'ils peuvent utiliser

  - vérifiez que vous avez une résolution de noms qui fonctionne avec `dig`
```
[logards@localhost ~]$ dig amazon.com

; <<>> DiG 9.16.23-RH <<>> amazon.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42135
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;amazon.com.                    IN      A

;; ANSWER SECTION:
amazon.com.             37      IN      A       52.94.236.248
amazon.com.             37      IN      A       54.239.28.85
amazon.com.             37      IN      A       205.251.242.103

;; Query time: 26 msec
;; SERVER: 10.0.2.3#53(10.0.2.3)
;; WHEN: Mon Oct 24 11:56:58 CEST 2022
;; MSG SIZE  rcvd: 87
```
```
[logards@localhost ~]$ ping google.com
PING google.com (142.250.179.78) 56(84) bytes of data.
64 bytes from par21s19-in-f14.1e100.net (142.250.179.78): icmp_seq=1 ttl=61 time=26.5 ms
64 bytes from par21s19-in-f14.1e100.net (142.250.179.78): icmp_seq=2 ttl=61 time=26.2 ms
64 bytes from par21s19-in-f14.1e100.net (142.250.179.78): icmp_seq=3 ttl=61 time=27.0 ms
64 bytes from par21s19-in-f14.1e100.net (142.250.179.78): icmp_seq=4 ttl=61 time=25.8 ms
^C
--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3007ms
rtt min/avg/max/mdev = 25.822/26.390/26.989/0.427 ms
```
  - puis avec un `ping` vers un nom de domaine

🌞**Analyse de trames**

- effectuez un `ping 8.8.8.8` depuis `john`
- capturez le ping depuis `john` avec `tcpdump`
- analysez un ping aller et le retour qui correspond et mettez dans un tableau :

| ordre | type trame | IP source          | MAC source              | IP destination | MAC destination |     |
|-------|------------|--------------------|-------------------------|----------------|-----------------|-----|
| 1     | ping       | `marcel` `10.3.1.12` | `marcel` `08:00:27:7e:00:07` | `8.8.8.8`      | `08:00:27:7a:89:aa`               |     |
| 2     | pong       | `8.8.8.8`               | `08:00:27:7a:89:aa`                     |`marcel` `10.3.1.12` | `marcel` `08:00:27:7e:00:07`

🦈 **Capture réseau `tp3_routage_internet.pcapng`**
[tp3_routage_internet](./tp3_routage_internet_essai.pcap)

## III. DHCP

On reprend la config précédente, et on ajoutera à la fin de cette partie une 4ème machine pour effectuer des tests.

| Machine  | `10.3.1.0/24`              | `10.3.2.0/24` |
|----------|----------------------------|---------------|
| `router` | `10.3.1.254`               | `10.3.2.254`  |
| `john`   | `10.3.1.11`                | no            |
| `bob`    | oui mais pas d'IP statique | no            |
| `marcel` | no                         | `10.3.2.12`   |

```schema
   john               router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └─┬─┘    └─────┘    └───┘    └─────┘
   john        │
  ┌─────┐      │
  │     │      │
  │     ├──────┘
  └─────┘
```

### 1. Mise en place du serveur DHCP

🌞**Sur la machine `john`, vous installerez et configurerez un serveur DHCP** (go Google "rocky linux dhcp server").

- installation du serveur sur `john`
- créer une machine `bob`
- faites lui récupérer une IP en DHCP à l'aide de votre serveur
```
[logards@localhost ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ab:78:4c brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.1/24 brd 10.3.1.255 scope global dynamic noprefixroute enp0s8
       valid_lft 607sec preferred_lft 607sec
    inet6 fe80::a00:27ff:feab:784c/64 scope link 
       valid_lft forever preferred_lft forever
```

> Il est possible d'utilise la commande `dhclient` pour forcer à la main, depuis la ligne de commande, la demande d'une IP en DHCP, ou renouveler complètement l'échange DHCP (voir `dhclient -h` puis call me et/ou Google si besoin d'aide).

🌞**Améliorer la configuration du DHCP**

- ajoutez de la configuration à votre DHCP pour qu'il donne aux clients, en plus de leur IP :
  - une route par défaut
  - un serveur DNS à utiliser
- récupérez de nouveau une IP en DHCP sur `bob` pour tester :
  - `marcel` doit avoir une IP
    - vérifier avec une commande qu'il a récupéré son IP
```
    [logards@localhost ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ab:78:4c brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.1/24 brd 10.3.1.255 scope global dynamic noprefixroute enp0s8
       valid_lft 631sec preferred_lft 631sec
    inet 10.3.1.2/24 brd 10.3.1.255 scope global secondary dynamic enp0s8
       valid_lft 880sec preferred_lft 880sec
    inet6 fe80::a00:27ff:feab:784c/64 scope link 
       valid_lft forever preferred_lft forever
```

    - vérifier qu'il peut `ping` sa passerelle
```
[logards@localhost ~]$ ping 10.3.1.254
PING 10.3.1.254 (10.3.1.254) 56(84) bytes of data.
64 bytes from 10.3.1.254: icmp_seq=1 ttl=64 time=0.632 ms
64 bytes from 10.3.1.254: icmp_seq=2 ttl=64 time=0.851 ms
64 bytes from 10.3.1.254: icmp_seq=3 ttl=64 time=1.01 ms
64 bytes from 10.3.1.254: icmp_seq=4 ttl=64 time=1.01 ms
^C
--- 10.3.1.254 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 0.632/0.874/1.008/0.153 ms
```
  - il doit avoir une route par défaut
    - vérifier la présence de la route avec une commande
```
[logards@localhost ~]$ ip route show
default via 10.3.1.254 dev enp0s8 
default via 10.3.1.254 dev enp0s8 proto dhcp src 10.3.1.1 metric 100 
10.3.1.0/24 dev enp0s8 proto kernel scope link src 10.3.1.1 metric 100 
```
    - vérifier que la route fonctionne avec un `ping` vers une IP
```
[logards@localhost ~]$ ping 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=1.35 ms
64 bytes from 10.3.2.12: icmp_seq=2 ttl=63 time=1.86 ms
64 bytes from 10.3.2.12: icmp_seq=3 ttl=63 time=1.77 ms
64 bytes from 10.3.2.12: icmp_seq=4 ttl=63 time=2.07 ms
^C
--- 10.3.2.12 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 1.351/1.761/2.067/0.260 ms
```
  - il doit connaître l'adresse d'un serveur DNS pour avoir de la résolution de noms
    - vérifier avec la commande `dig` que ça fonctionne
```
[logards@localhost ~]$ dig frandroid.fr

; <<>> DiG 9.16.23-RH <<>> frandroid.fr
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34526
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;frandroid.fr.                  IN      A

;; ANSWER SECTION:
frandroid.fr.           10800   IN      A       199.59.243.222

;; Query time: 51 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Tue Oct 25 14:35:08 CEST 2022
;; MSG SIZE  rcvd: 57
```
    - vérifier un `ping` vers un nom de domaine
```
[logards@localhost ~]$ ping frandroid.fr
PING frandroid.fr (199.59.243.222) 56(84) bytes of data.
64 bytes from 199.59.243.222 (199.59.243.222): icmp_seq=1 ttl=61 time=29.2 ms
64 bytes from 199.59.243.222 (199.59.243.222): icmp_seq=2 ttl=61 time=28.5 ms
64 bytes from 199.59.243.222 (199.59.243.222): icmp_seq=3 ttl=61 time=28.5 ms
64 bytes from 199.59.243.222 (199.59.243.222): icmp_seq=4 ttl=61 time=25.9 ms
^C
--- frandroid.fr ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3007ms
rtt min/avg/max/mdev = 25.935/28.041/29.220/1.250 ms
```

### 2. Analyse de trames

🌞**Analyse de trames**

- lancer une capture à l'aide de `tcpdump` afin de capturer un échange DHCP

- demander une nouvelle IP afin de générer un échange DHCP
- exportez le fichier `.pcapng`
[tp3_dhcp.pcapng](./tp3_dhcp.pcap)

🦈 **Capture réseau `tp3_dhcp.pcapng`**
