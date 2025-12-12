## idea jest prosta, w zaleznosci od osiagalnosci danego adresu w internecie - w naszym wypadku DNS google 8.8.8.8 przerzucamy priorytet VRRP z jedneo na drugi router.
Arista nie ma domyslnie mechanismu trackujacego ktory pozwala wykorzystac odpowiednik IP SLA Cisco, a tracking w VRRP moze reduckowac priorytet VRRP tylko w przypadku gdy monitorowany interejs pojdzie w doł.
Zatem nasz lab scenario definujemy w kilku krokach:

1. na routerach VRRP1 i VRRP2 konfigurujemy VRRP jak ponizej:

```bash
VRRP1(config-router-bgp)#   sh run interfaces ethernet 4
interface Ethernet4
   description to_LAN
   no switchport
   ip address 192.168.0.1/24
   vrrp 1 priority-level 250
   vrrp 1 ipv4 192.168.0.250
   vrrp 1 ipv4 version 3
```

i sprawdzamy faktyczny status:

```bash
VRRP1(config-router-bgp)#sh vrrp brief 
Interface VRF        ID  Ver Pri Time  State  Last Transition     VR IP Addresses 
--------- ---------- --- --- --- ----- ------ ------------------- ---------------
Et4       default    1   3   250 3020  Master 00:02:28 ago        192.168.0.250   
VRRP1(config-router-bgp)#
```

2. nastepnie sprawdzamy connectivity monitor oraz event-handlers:

```bash
monitor connectivity
   interval 5
   no shutdown
   interface set int_to_ISP2 Ethernet2
   !
   host DNS
      description "ping do DNS google 8.8.8.8"
      local-interfaces int_to_ISP2 address-only
      ip 8.8.8.8
!
event-handler DNS_google_DOWN
   action bash Cli -p15 -c $'enable
 configure
 interface Ethernet 4
 vrrp 1 priority-level 50
 end'
   !
   trigger on-logging
      regex DNS.*8.8.8.8* is unreachable .*Ethernet2
!
event-handler DNS_google_UP
   action bash Cli -p15 -c $'enable
 configure
 interface Ethernet 4
 vrrp 1 priority-level 150
 end'
   !
   trigger on-logging
      regex DNS.*8.8.8.8* is reachable .*Ethernet2
!
```

```bash
VRRP1#sh monitor connectivity 

VRF: default
Host: DNS
Payload size: 56
ICMP ping count: 5
Description: "ping do DNS google 8.8.8.8"
Network statistics:
IP Address Local Interface  Latency  Jitter Packet Loss Probe Error
---------- --------------- -------- ------- ----------- -----------
8.8.8.8    Ethernet1       0.229 ms 0.02 ms          0% n/a
```

Oraz routing:

```bash
VRRP1#sh ip route 8.8.8.8
[...]
via 10.10.10.1, Ethernet1
```

3. Symulacja niedostepnosci DNS – shutdown Lo88 na ISP1:

PRZED:

```bash
VRRP1# sh ip bgp 
[...]
 * >      8.8.8.8/32             10.10.10.1
 *        8.8.8.8/32             192.168.0.2
```

PO:

```bash
ISP1(config-if-Lo88)#shutdown
```

Efekt:

```bash
VRRP1# sh ip bgp
 * >      8.8.8.8/32             192.168.0.2
```

Monitor:

```bash
VRRP1#sh monitor connectivity 
8.8.8.8    Ethernet1           n/a    n/a        100% Network is unreachable
```

4. Akcja przełączania:

```bash
Dec 12 12:49:17 VRRP1 ConnectivityMonitor: Host DNS (8.8.8.8) is reachable [...]
Dec 12 12:49:37 VRRP1 EventMgr: Event handler DNS_google_UP was activated
Dec 12 12:49:40 VRRP1 Fhrp: Ethernet4 Grp 1 state Backup -> Master
```

VRRP:

```bash
VRRP1#sh vrrp brief 
Et4 default 1 3 50 Backup
```

5. Powrót:

```bash
ISP1(config-if-Lo88)#no shutdown
```

```bash
VRRP1#sh vrrp brief 
Et4 default 1 3 250 Master
```

Enjoy!
