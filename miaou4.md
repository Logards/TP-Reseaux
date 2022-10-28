

🌞 **Déterminez, pour ces 5 applications, si c'est du TCP ou de l'UDP**

- avec Wireshark, on va faire les chirurgiens réseau
- déterminez, pour chaque application :
  - IP et port du serveur auquel vous vous connectez
  - le port local que vous ouvrez pour vous connecter

> Dès qu'on se connecte à un serveur, notre PC ouvre un port random. Une fois la connexion TCP ou UDP établie, entre le port de notre PC et le port du serveur qui est en écoute, on parle de tunnel TCP ou de tunnel UDP.


> Aussi, TCP ou UDP ? Comment le client sait ? Il sait parce que le serveur a décidé ce qui était le mieux pour tel ou tel type de trafic (un jeu, une page web, etc.) et que le logiciel client est codé pour utiliser TCP ou UDP en conséquence.

🌞 **Demandez l'avis à votre OS**

- votre OS est responsable de l'ouverture des ports, et de placer un programme en "écoute" sur un port
- il est aussi responsable de l'ouverture d'un port quand une application demande à se connecter à distance vers un serveur
- bref il voit tout quoi
- utilisez la commande adaptée à votre OS pour repérer, dans la liste de toutes les connexions réseau établies, la connexion que vous voyez dans Wireshark, pour chacune des 5 applications
```
commande utilisée : sudo ss -tunp
```

```
spotify : 45081                         35.186.224.25:443 
```
[spotify](./spotify.pcapng)
```
prospect mail : 39302                          40.101.137.2:443  
```
[prospect-mail](./prospect%20mail.pcapng)
```
discord : 51976                       162.159.133.234:443  
```
[discord](./discord.pcapng)

```
firefox :   50238                           52.89.15.44:443     
``` 
[firefox](./firefox.pcapng)
```
vsc : 59778                        152.199.19.160:443   
```
[vsc](./code)

🦈🦈🦈🦈🦈 **Bah ouais, captures Wireshark à l'appui évidemment.** Une capture pour chaque application, qui met bien en évidence le trafic en question.


🌞 **Examinez le trafic dans Wireshark**

- **déterminez si SSH utilise TCP ou UDP**
```
TCP
```
  - pareil réfléchissez-y deux minutes, logique qu'on utilise pas UDP non ?
- **repérez le *3-Way Handshake* à l'établissement de la connexion**
  - c'est le `SYN` `SYNACK` `ACK`
- **repérez du trafic SSH**
- **repérez le FIN ACK à la fin d'une connexion**
[je sèche je n'ai plus de nom](./ZAWARUDO.pcapng)
- entre le *3-way handshake* et l'échange `FIN`, c'est juste une bouillie de caca chiffré, dans un tunnel TCP

> **SUR WINDOWS, pour cette étape uniquement**, utilisez Git Bash et PAS Powershell. Avec Powershell il sera très difficile d'observer le FIN ACK.

🌞 **Demandez aux OS**

- repérez, avec une commande adaptée (`netstat` ou `ss`), la connexion SSH depuis votre machine
```
tcp             ESTAB                0                0                                   10.4.1.1:53270                             10.4.1.11:22              users:(("ssh",pid=5162,fd=3))                       
```

- ET repérez la connexion SSH depuis votre VM
```
tcp                  ESTAB                0                     0                                          10.4.1.11:22                                       10.4.1.1:53270                 users:(("sshd",pid=838,fd=4),("sshd",pid=832,fd=4))
```

🦈 **Je veux une capture clean avec le 3-way handshake, un peu de trafic au milieu et une fin de connexion**$


🌞 **Dans le rendu, je veux**

- un `cat` des fichiers de conf
```
[logards@leolio ~]$ sudo !!
sudo cat  /etc/named.conf 
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        listen-on port 53 { 127.0.0.1; any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { localhost; any; };
        allow-query-cache { localhost; any; };
        /* 
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable 
           recursion. 
         - If your recursive DNS server has a public IP address, you MUST enable access 
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification 
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface 
        */
        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

# référence vers notre fichier de zone
zone "tp4.b1" IN {
     type master;
     file "tp4.b1.db";
     allow-update { none; };
     allow-query {any; };
};
# référence vers notre fichier de zone inverse
zone "1.4.10.in-addr.arpa" IN {
     type master;
     file "tp4.b1.rev";
     allow-update { none; };
     allow-query { any; };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```
```
[logards@leolio ~]$ sudo cat  /var/named/tp4.b1.db
$TTL 86400
@ IN SOA dns-server.tp4.b1. admin.tp4.b1. (
    2019061800 ;Serial
    3600 ;Refresh
    1800 ;Retry
    604800 ;Expire
    86400 ;Minimum TTL
)

; Infos sur le serveur DNS lui même (NS = NameServer)
@ IN NS dns-server.tp4.b1.

; Enregistrements DNS pour faire correspondre des noms à des IPs
dns-server IN A 10.4.1.201
node1      IN A 10.4.1.11
```
```
[logards@leolio ~]$ sudo cat /var/named/tp4.b1.rev
$TTL 86400
@ IN SOA dns-server.tp4.b1. admin.tp4.b1. (
    2019061800 ;Serial
    3600 ;Refresh
    1800 ;Retry
    604800 ;Expire
    86400 ;Minimum TTL
)

; Infos sur le serveur DNS lui même (NS = NameServer)
@ IN NS dns-server.tp4.b1.

;Reverse lookup for Name Server
201 IN PTR dns-server.tp4.b1.
11 IN PTR node1.tp4.b1.
```
- un `systemctl status named` qui prouve que le service tourne bien
```
[logards@leolio ~]$ systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
     Active: active (running) since Fri 2022-10-28 16:33:40 CEST; 7min ago
   Main PID: 2504 (named)
      Tasks: 5 (limit: 5905)
     Memory: 17.7M
        CPU: 96ms
     CGroup: /system.slice/named.service
             └─2504 /usr/sbin/named -u named -c /etc/named.conf

Oct 28 16:33:40 leolio.lab.ingesup named[2504]: network unreachable resolving './NS/IN': 2001:500:1::53#53
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: zone 1.0.0.127.in-addr.arpa/IN: loaded serial 0
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: zone tp4.b1/IN: loaded serial 2019061800
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: zone localhost/IN: loaded serial 0
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: zone 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa/IN: loaded serial 0
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: all zones loaded
Oct 28 16:33:40 leolio.lab.ingesup systemd[1]: Started Berkeley Internet Name Domain (DNS).
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: running
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: managed-keys-zone: Initializing automatic trust anchor management for zone '.'; DNSKEY ID 20326 is now trusted, waiving the normal 30-day waiting period.
Oct 28 16:33:40 leolio.lab.ingesup named[2504]: resolver priming query complete
```
- une commande `ss` qui prouve que le service écoute bien sur un port
```
udp                  UNCONN                0                     0                                         10.4.1.201:53                                        0.0.0.0:*                                          
```

🌞 **Ouvrez le bon port dans le firewall**

- grâce à la commande `ss` vous devrez avoir repéré sur quel port tourne le service
  - vous l'avez écrit dans la conf aussi toute façon :)
- ouvrez ce port dans le firewall de la machine `dns-server.tp4.b1` (voir le mémo réseau Rocky)

```
[logards@leolio ~]$ sudo firewall-cmd --add-port=53/udp --permanent
```


```
[logards@leolio ~]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s8
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 53/udp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

## 3. Test

🌞 **Sur la machine `node1.tp4.b1`**

- configurez la machine pour qu'elle utilise votre serveur DNS quand elle a besoin de résoudre des noms
- assurez vous que vous pouvez :
  - résoudre des noms comme `node1.tp4.b1` et `dns-server.tp4.b1`
```
  [logards@gon1 ~]$ dig node1.tp4.b1 @10.4.1.201

; <<>> DiG 9.16.23-RH <<>> node1.tp4.b1 @10.4.1.201
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1903
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: c1e488ea229359c301000000635bef6fe0d4f396c715d0fd (good)
;; QUESTION SECTION:
;node1.tp4.b1.                  IN      A

;; ANSWER SECTION:
node1.tp4.b1.           86400   IN      A       10.4.1.11

;; Query time: 3 msec
;; SERVER: 10.4.1.201#53(10.4.1.201)
;; WHEN: Fri Oct 28 17:04:15 CEST 2022
;; MSG SIZE  rcvd: 85
```
```
[logards@gon1 ~]$ dig dns-server.tp4.b1

; <<>> DiG 9.16.23-RH <<>> dns-server.tp4.b1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 48102
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;dns-server.tp4.b1.             IN      A

;; AUTHORITY SECTION:
.                       7110    IN      SOA     a.root-servers.net. nstld.verisign-grs.com. 2022102800 1800 900 604800 86400

;; Query time: 3 msec
;; SERVER: 10.0.2.3#53(10.0.2.3)
;; WHEN: Fri Oct 28 17:04:30 CEST 2022
;; MSG SIZE  rcvd: 121
```

  - mais aussi des noms comme `www.google.com`
```
[logards@gon1 ~]$ dig www.google.com

; <<>> DiG 9.16.23-RH <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56480
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;www.google.com.                        IN      A

;; ANSWER SECTION:
www.google.com.         259     IN      A       216.58.214.164

;; Query time: 24 msec
;; SERVER: 10.0.2.3#53(10.0.2.3)
;; WHEN: Fri Oct 28 17:04:38 CEST 2022
;; MSG SIZE  rcvd: 59
```

🌞 **Sur votre PC**

- utilisez une commande pour résoudre le nom `node1.tp4.b1` en utilisant `10.4.1.201` comme serveur DNS
```
[bastien@fedora ~]$ dig node1.tp4.b1@10.4.1.201

; <<>> DiG 9.16.33-RH <<>> node1.tp4.b1@10.4.1.201
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 52716
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;node1.tp4.b1\@10.4.1.201.      IN      A

;; AUTHORITY SECTION:
.                       86367   IN      SOA     a.root-servers.net. nstld.verisign-grs.com. 2022102800 1800 900 604800 86400

;; Query time: 54 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Fri Oct 28 17:08:39 CEST 2022
;; MSG SIZE  rcvd: 127

```

> Le fait que votre serveur DNS puisse résoudre un nom comme `www.google.com`, ça s'appelle la récursivité et c'est activé avec la ligne `recursion yes;` dans le fichier de conf.

[fin](./fin%20the%20end.pcapng)
🦈 **Capture d'une requête DNS vers le nom `node1.tp4.b1` ainsi que la réponse**
