
## 3. Setup topologie 1

🌞 **Commençons simple**

- définissez les IPs statiques sur les deux VPCS
- `ping` un VPCS depuis l'autre
pc1
```
ip 10.5.1.1 
```
pc2
```
ip 10.5.1.2 
```
```
PC2> ping 10.5.1.1

84 bytes from 10.5.1.1 icmp_seq=1 ttl=64 time=0.160 ms
84 bytes from 10.5.1.1 icmp_seq=2 ttl=64 time=0.672 ms
84 bytes from 10.5.1.1 icmp_seq=3 ttl=64 time=0.655 ms
84 bytes from 10.5.1.1 icmp_seq=4 ttl=64 time=0.684 ms
84 bytes from 10.5.1.1 icmp_seq=5 ttl=64 time=0.523 ms
```

> Jusque là, ça devrait aller. Noter qu'on a fait aucune conf sur le switch. Tant qu'on ne fait rien, c'est une bête multiprise.

# II. VLAN

🌞 **Adressage**

- définissez les IPs statiques sur tous les VPCS
- vérifiez avec des `ping` que tout le monde se ping
```
pc1 to pc2
PC1> ping 10.5.10.2

84 bytes from 10.5.10.2 icmp_seq=1 ttl=64 time=0.434 ms
84 bytes from 10.5.10.2 icmp_seq=2 ttl=64 time=0.660 ms
84 bytes from 10.5.10.2 icmp_seq=3 ttl=64 time=0.543 ms
84 bytes from 10.5.10.2 icmp_seq=4 ttl=64 time=0.414 ms
84 bytes from 10.5.10.2 icmp_seq=5 ttl=64 time=0.518 ms

pc1 to pc3
PC1> ping 10.5.10.3

84 bytes from 10.5.10.3 icmp_seq=1 ttl=64 time=0.257 ms
84 bytes from 10.5.10.3 icmp_seq=2 ttl=64 time=0.355 ms
84 bytes from 10.5.10.3 icmp_seq=3 ttl=64 time=0.419 ms
84 bytes from 10.5.10.3 icmp_seq=4 ttl=64 time=0.354 ms

pc2 to pc3

PC2> ping 10.5.10.3

84 bytes from 10.5.10.3 icmp_seq=1 ttl=64 time=0.251 ms
84 bytes from 10.5.10.3 icmp_seq=2 ttl=64 time=0.404 ms
84 bytes from 10.5.10.3 icmp_seq=3 ttl=64 time=0.490 ms
84 bytes from 10.5.10.3 icmp_seq=4 ttl=64 time=0.537 ms
```

🌞 **Configuration des VLANs**

- référez-vous [à la section VLAN du mémo Cisco](../../cours/memo/memo_cisco.md#8-vlan)
- déclaration des VLANs sur le switch `sw1`
- ajout des ports du switches dans le bon VLAN (voir [le tableau d'adressage de la topo 2 juste au dessus](#2-adressage-topologie-2))
  - ici, tous les ports sont en mode *access* : ils pointent vers des clients du réseau

🌞 **Vérif**

- `pc1` et `pc2` doivent toujours pouvoir se ping
```
PC1> ping 10.5.10.2

84 bytes from 10.5.10.2 icmp_seq=1 ttl=64 time=0.224 ms
84 bytes from 10.5.10.2 icmp_seq=2 ttl=64 time=0.411 ms
```
```
PC3> ping 10.5.10.1

host (10.5.10.1) not reachable
```
- `pc3` ne ping plus personne

Il assurera son job de routeur traditionnel : router entre deux réseaux. Sauf qu'en plus, il gérera le changement de VLAN à la volée.

🌞 **Adressage**

- définissez les IPs statiques sur toutes les machines **sauf le *routeur***
```
ip 10.5.10.1
ip 10.5.10.2
ip 10.5.20.1
```


🌞 **Configuration des VLANs**

```
sw1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
sw1(config)#vlan 10
sw1(config-vlan)#name clients
sw1(config-vlan)#exit
sw1(config)#vlan 20
sw1(config-vlan)#name admins
sw1(config-vlan)#exit
sw1(config)#vlan 30
sw1(config-vlan)#name servers
sw1(config-vlan)#exit
sw1(config)#exit

sw1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
sw1(config)#interface Ethernet0/2
sw1(config-if)#switchport mode access
sw1(config-if)#switchport access vlan 20
sw1(config-if)#exit
sw1(config)#interface Ethernet0/3
sw1(config-if)#switchport mode access
sw1(config-if)#switchport access vlan 30
sw1(config-if)#exit
sw1(config)#

sw1#show vlan br 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et1/1, Et1/2, Et1/3, Et2/0
                                                Et2/1, Et2/2, Et2/3, Et3/0
                                                Et3/1, Et3/2, Et3/3
10   clients                          active    Et0/0, Et0/1
20   admins                           active    Et0/2
30   servers                          active    Et0/3
1002 fddi-default                     act/unsup 
1003 token-ring-default               act/unsup 
1004 fddinet-default                  act/unsup 
1005 trnet-default                    act/unsup 

```




---

🌞 **Config du *routeur***

- attribuez ses IPs au *routeur*
  - 3 sous-interfaces, chacune avec son IP et un VLAN associé

```
# conf t

(config)# interface fastEthernet 0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip addr 10.5.10.254 255.255.255.0 
R1(config-subif)# exit

(config)# interface fastEthernet 0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip addr 10.5.20.254 255.255.255.0 
R1(config-subif)# exit

(config)# interface fastEthernet 0/0.30
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip addr 10.5.30.254 255.255.255.0 
R1(config-subif)# exit
```


🌞 **Vérif**

- tout le monde doit pouvoir ping le routeur sur l'IP qui est dans son réseau
- en ajoutant une route vers les réseaux, ils peuvent se ping entre eux
  - ajoutez une route par défaut sur les VPCS
```
ip 10.5.10.1/24 10.5.10.254 
ip 10.5.10.2/24 10.5.10.254 
ip 10.5.20.1/24 10.5.20.254 

et pour la machine ajout dans le fichier de ifcfg-enp0s8
GATEWAY = 10.5.30.254
```
  - ajoutez une route par défaut sur la machine virtuelle
  - testez des `ping` entre les réseaux
```
VPCS> ping 10.5.10.2

84 bytes from 10.5.10.2 icmp_seq=1 ttl=64 time=0.257 ms
84 bytes from 10.5.10.2 icmp_seq=2 ttl=64 time=1.029 ms
84 bytes from 10.5.10.2 icmp_seq=3 ttl=64 time=1.055 ms
84 bytes from 10.5.10.2 icmp_seq=4 ttl=64 time=0.431 ms

VPCS> ping 10.5.20.1

10.5.20.1 icmp_seq=1 timeout
84 bytes from 10.5.20.1 icmp_seq=2 ttl=63 time=15.289 ms
84 bytes from 10.5.20.1 icmp_seq=3 ttl=63 time=18.632 ms
84 bytes from 10.5.20.1 icmp_seq=4 ttl=63 time=16.875 ms
84 bytes from 10.5.20.1 icmp_seq=5 ttl=63 time=18.295 ms

VPCS> ping 10.5.30.1

10.5.30.1 icmp_seq=1 timeout
84 bytes from 10.5.30.1 icmp_seq=2 ttl=63 time=15.311 ms
84 bytes from 10.5.30.1 icmp_seq=3 ttl=63 time=23.950 ms
84 bytes from 10.5.30.1 icmp_seq=4 ttl=63 time=17.585 ms
```


# IV. NAT


## 3. Setup topologie 4

🌞 **Ajoutez le noeud Cloud à la topo**

- branchez à `eth1` côté Cloud
- côté routeur, il faudra récupérer un IP en DHCP (voir [le mémo Cisco](../../cours/memo/memo_cisco.md))
```
R1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#interface fastEthernet0/1

R1(config-if)#ip address dhcp
R1(config-if)#no shut
R1(config-if)#
*Mar  1 00:29:57.287: %LINK-3-UPDOWN: Interface FastEthernet0/1, changed state to up
*Mar  1 00:29:58.287: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to up
R1(config-if)#exit
R1(config)#exit
R1#
*Mar  1 00:30:04.679: %SYS-5-CONFIG_I: Configured from console by console
R1#show ip int br
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            unassigned      YES NVRAM  up                    up      
FastEthernet0/0.10         10.5.10.254     YES NVRAM  up                    up      
FastEthernet0/0.20         10.5.20.254     YES NVRAM  up                    up      
FastEthernet0/0.30         10.5.30.254     YES NVRAM  up                    up      
FastEthernet0/1            unassigned      YES DHCP   up                    up      
FastEthernet1/0            unassigned      YES NVRAM  administratively down down    
FastEthernet2/0            unassigned      YES NVRAM  administratively down down    
R1#
```
- vous devriez pouvoir `ping 1.1.1.1`


