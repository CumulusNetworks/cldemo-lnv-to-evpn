# Cumulus Linux LNV to EVPN Migration
![Reference Topology](./lnv-to-evpn-topo.png "Topology")

Welcome to the Cumulus Linux LNV to EVPN Migration demo. Using the [Cumulus reference topology](https://github.com/CumulusNetworks/cldemo-vagrant), this demo will walk through the steps to migrate your LNV controlled VXLAN to the de facto standard using BGP EVPN.  LNV (the vxfld package) will be depricated in Cumulus Linux 4.x in favor of using EVPN to manage VXLAN overlays.

This demo will start you with an LNV controlled VXLAN topology, then step through the process of converting to EVPN using BGP. One way will be through using ad-hoc ansible commands and NCLU.  The other way will be through an Ansible playbook that directly changes the config files and restarts the associated services.  Below are a few additional notes and details of this topology:

1. Using a [centralized routing](https://cumulusnetworks.com/blog/vxlan-designs-part-1/) model.  Routing between VXLANs is done at the exit leafs.  The leaf switches exit01 and exit02 have the SVIs and provide the first hop redundancy.
2. VXLAN is in active-active mode on the MLAG enabled leafs.  The leaf switches are in pairs to provide an MLAG bond to the servers (simulating a rack).  Inbound VXLAN traffic from the rest of the fabric to the pair is addressed to the clagd-vxlan-anycast-ip IP address.  We'll see this clagd-vxlan-anycast-ip address instead of the individual loopbacks when looking at some LNV and BGP EVPN output
3. The LNV Service Nodes (vxsnd) are running in [anycast mode](https://docs.cumulusnetworks.com/display/DOCS/Lightweight+Network+Virtualization+Overview#LightweightNetworkVirtualizationOverview-ScaleLNVbyLoadBalancingwithAnycast) on both spine01 and spine02.

Need to also link to the companion whitepaper:

For more information about the reference topology and other demos based on this topology, head on over to: https://github.com/CumulusNetworks/cldemo-vagrant

### Provisioning the Topology

After cloning (or downloading/extracting the .zip), change into the new directory named "cldemo-lnv-to-evpn." From there, bring up the oob-mgmt-server and oob-mgmt-switch first.  Brining the oob-mgmt devices up first helps make sure that the DHCP server on the oob-mgmt-server is up and reliably ready to hand out IP addresses to the rest of the network when we provision it all with the last step, 'vagrant up'

    git clone https://github.com/cumulusnetworks/cldemo-lnv-to-evpn
    cd cldemo-lnv-to-evpn
    vagrant up oob-mgmt-server oob-mgmt-switch
    vagrant up

### Logging in
Once Vagrant has finished all of its provisioning, log into the oob-mgmt-server.  We'll be able to use ansible from the oob-mgmt-server to setup the rest of the demo and be able to jump into the other nodes in the topology from here.

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
### Setting up the demo
After logging into the oob-mgmt-server, change directories to the 'lnv-to-evpn' folder.  From there, run the ansible playbook named run_demo.yml

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible-playbook run_demo.yml 

PLAY [host] *************************************************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************************************
```

test

### 

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
