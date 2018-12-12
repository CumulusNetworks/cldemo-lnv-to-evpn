# Cumulus Linux LNV to EVPN Migration
![Reference Topology](./documentation/cldemo_topology.png "Reference Topology")

Welcome to the Cumulus Linux LNV to EVPN Migration demo. Using the [Cumulus reference topology](https://github.com/CumulusNetworks/cldemo-vagrant), this demo will walk through the steps to migrate your VXLAN control plane to the de facto standard using BGP EVPN.  LNV (the vxfld package) will be depricated in Cumulus Linux 4.x in favor of using EVPN to manage the VXLAN overlay. 

Link to the companion whitepaper:

For more information about the reference topology and other demos based on this topology, head on over to: https://github.com/CumulusNetworks/cldemo-vagrant


### Provisioning the Topology

    git clone https://github.com/cumulusnetworks/cldemo-lnv-to-evpn
    cd cldemo-lnv-to-evpn
    vagrant up oob-mgmt-server oob-mgmt-switch
    vagrant up

### Logging in

```
$ vagrant ssh oob-mgmt-server
                                                 _
      _______   x x x                           | |
 ._  <_______~ x X x   ___ _   _ _ __ ___  _   _| |_   _ ___
(' \  ,' || `,        / __| | | | '_ ` _ \| | | | | | | / __|
 `._:^   ||   :>     | (__| |_| | | | | | | |_| | | |_| \__ \
     ^T~~~~~~T'       \___|\__,_|_| |_| |_|\__,_|_|\__,_|___/
     ~"     ~"


############################################################################
#
#                     Out Of Band Management Station
#
############################################################################
cumulus@oob-mgmt-server:~$
```

test


```cumulus@oob-mgmt-server:~$ ansible server01 -a 'traceroute -n 10.2.4.104'
server01 | SUCCESS | rc=0 >>
traceroute to 10.2.4.104 (10.2.4.104), 30 hops max, 60 byte packets
 1  10.1.3.1  2.161 ms  2.106 ms  2.071 ms
 2  10.2.4.104  6.210 ms  6.165 ms *

cumulus@oob-mgmt-server:~$
```

---

>©2018 Cumulus Networks. CUMULUS, the Cumulus Logo, CUMULUS NETWORKS, and the Rocket Turtle Logo 
(the “Marks”) are trademarks and service marks of Cumulus Networks, Inc. in the U.S. and other 
countries. You are not permitted to use the Marks without the prior written consent of Cumulus 
Networks. The registered trademark Linux® is used pursuant to a sublicense from LMI, the exclusive 
licensee of Linus Torvalds, owner of the mark on a world-wide basis. All other marks are used under 
fair use or license from their respective owners.

For further details please see: [cumulusnetworks.com](http://www.cumulusnetworks.com)
