# Cumulus Linux LNV to EVPN Migration Demo
![Reference Topology](./lnv-to-evpn-topo.png "Topology")

Welcome to the Cumulus Linux LNV to EVPN Migration demo. Using the [Cumulus reference topology](https://github.com/CumulusNetworks/cldemo-vagrant), this demo will walk through the steps to migrate your LNV controlled VXLAN to the de facto standard using BGP EVPN.  LNV (the vxfld package) will be depricated in Cumulus Linux 4.x in favor of using EVPN to manage VXLAN overlays.

This demo will start you with an LNV controlled VXLAN topology, then step through the process of converting to EVPN using BGP. One way will be through using ad-hoc ansible commands and NCLU.  The other way will be through an Ansible playbook that directly changes the config files and restarts the associated services.  Below are a few additional notes and details of this topology:

1. Using a [centralized routing](https://cumulusnetworks.com/blog/vxlan-designs-part-1/) model. Routing between VXLANs is done at the exit leafs.  The leaf switches exit01 and exit02 have the SVIs and provide the first hop redundancy.
2. VXLAN is in active-active mode on the MLAG enabled leafs.  The leaf switches are in pairs to provide an MLAG bond to the servers (simulating a rack).  Inbound VXLAN traffic from the rest of the fabric to the pair is addressed to the clagd-vxlan-anycast-ip IP address.  We'll see this clagd-vxlan-anycast-ip address instead of the individual loopbacks when looking at some LNV and BGP EVPN output
3. The LNV Service Nodes (vxsnd) are running in [anycast mode](https://docs.cumulusnetworks.com/display/DOCS/Lightweight+Network+Virtualization+Overview#LightweightNetworkVirtualizationOverview-ScaleLNVbyLoadBalancingwithAnycast) on both spine01 and spine02.

Need to also link to the companion whitepaper:

For more information about the reference topology and other demos based on this topology, head on over to: https://github.com/CumulusNetworks/cldemo-vagrant

## Outline

1. Deploy the topology
2. Log into the oob-mgmt-server
3. Run Ansible playbook run_demo.yml to setup the LNV scenario
4. Inspect the LNV Environment
5. Perform the migration to BGP EVPN
6. Verification

### Deploy the Topology

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

### Checking the LNV Environment

After the 'run_demo.yml' ansible playbook completes, we will have a functioning LNV controlled VXLAN topology.  Lets run a few commands to generate some traffic and illustrate the topology.  First, lets run a traceroute from server01 to server04 in the other subnet and in the other rack.  We can use ansible from the oob-mgmt-server to run this traceroute for us and return the result: 

```
cumulus@oob-mgmt-server:~$ ansible server01 -a 'traceroute -n 10.2.4.104'
server01 | SUCCESS | rc=0 >>
traceroute to 10.2.4.104 (10.2.4.104), 30 hops max, 60 byte packets
 1  10.1.3.1  2.161 ms  2.106 ms  2.071 ms
 2  10.2.4.104  6.210 ms  6.165 ms *

cumulus@oob-mgmt-server:~$
```

Next, lets confirm that LNV is running and   Again, we can use ansible to run an NCLU command on all of the network nodes at the same time.

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible network -a 'net show lnv'
spine01 | SUCCESS | rc=0 >>

  LNV Role           : Service Node
  Version            : 3
  Local IP           : 10.0.0.21
  Anycast IP         : 10.0.0.200
  UDP Data Port      : 4789
  UDP Control Port   : 10001
  Service Node Peers : 10.0.0.21, 10.0.0.22

  VNI  VTEP        Ageout
  ---  ----------  ------
  13   10.0.0.100      86 <- leaf01/02
       10.0.0.100      88 <- leaf01/02
       10.0.0.101      86 <- leaf03/04
       10.0.0.101      86 <- leaf03/04
       10.0.0.40       90 <- exit01/02
       10.0.0.40       90 <- exit01/02
  24   10.0.0.100      86
       10.0.0.100      88
       10.0.0.101      86
       10.0.0.101      86
       10.0.0.40       90
       10.0.0.40       90
<trimmed for brevity>
```
The output from the ad hoc ansible command will be separated by host.  You should see that the spines are LNV Role: Service Node and leafs/exit are LNV Role: VTEP. Remeber that with VXLAN active-active mode (clagd-vxlan-anycast-ip) we will see each VTEP register with it's anycast address.

Lastly, lets take a look at a bridge mac address table on one of the leafs that has the VXLAN VTEPs.  Your MAC addresses may differ from this example.  Notice that the linux bridge also learns the source VTEP IP address (TunnelDest) for MAC addresses that exist behind other VTEPs. 

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible leaf01 -a 'net show bridge macs'
leaf01 | SUCCESS | rc=0 >>

VLAN      Master  Interface  MAC                TunnelDest  State      Flags  LastSeen
--------  ------  ---------  -----------------  ----------  ---------  -----  --------
13        bridge  bond01     00:03:00:11:11:02                                00:00:36
13        bridge  bond01     02:03:00:11:11:01                                00:00:07
13        bridge  bond01     02:03:00:11:11:02                                00:01:46
13        bridge  vni-13     44:38:39:00:00:0c                                00:01:46
13        bridge  vni-13     44:38:39:00:00:4b                                00:01:46
13        bridge  vni-13     44:39:39:ff:00:13                                00:01:46
24        bridge  bond02     02:03:00:22:22:01                                00:00:07
24        bridge  bond02     02:03:00:22:22:02                                00:01:46
24        bridge  vni-24     00:03:00:44:44:02                                00:09:03
24        bridge  vni-24     44:38:39:00:00:0c                                00:01:46
24        bridge  vni-24     44:38:39:00:00:4b                                00:01:46
24        bridge  vni-24     44:39:39:ff:00:24                                00:01:46
untagged          vni-13     00:00:00:00:00:00  10.0.0.40   permanent  self   00:10:48 <- BUM traffic handling
untagged          vni-13     00:00:00:00:00:00  10.0.0.101  permanent  self   00:10:48 <- BUM traffic handling
untagged          vni-13     44:38:39:00:00:0c  10.0.0.40              self   00:09:02 <- Learned VXLAN MAC address
untagged          vni-13     44:38:39:00:00:4b  10.0.0.40              self   00:09:02 <- Learned VXLAN MAC address
untagged          vni-13     44:39:39:ff:00:13  10.0.0.40              self   00:10:03 <- Learned VXLAN MAC address
untagged          vni-24     00:00:00:00:00:00  10.0.0.40   permanent  self   00:10:48 <- BUM traffic handling
untagged          vni-24     00:00:00:00:00:00  10.0.0.101  permanent  self   00:10:48 <- BUM traffic handling
untagged          vni-24     00:03:00:44:44:02  10.0.0.101             self   00:09:03 <- Learned VXLAN MAC address
untagged          vni-24     44:38:39:00:00:0c  10.0.0.40              self   00:09:02 <- Learned VXLAN MAC address
untagged          vni-24     44:38:39:00:00:4b  10.0.0.40              self   00:09:02 <- Learned VXLAN MAC address
untagged          vni-24     44:39:39:ff:00:24  10.0.0.40              self   00:09:30 <- Learned VXLAN MAC address
untagged  bridge  bond01     44:38:39:00:00:03              permanent         00:10:51
untagged  bridge  bond02     44:38:39:00:00:14              permanent         00:10:51
untagged  bridge  peerlink   44:38:39:00:00:10              permanent         00:10:51
untagged  bridge  vni-13     9e:32:c5:25:6b:84              permanent         00:10:51
untagged  bridge  vni-24     6e:76:b8:14:da:2b              permanent         00:10:51

cumulus@oob-mgmt-server:~/lnv-to-evpn$ 
```

### Performing the migration

We'll perform this upgrade step by step using NCLU and ad hoc ansible commands.  Device groups that we'll be using are:

vtep - leaf01, leaf02, leaf03, leaf04, exit01, exit02<br>
spine - spine01, spine02

1. First lets get the new EVPN BGP address family configuration staged.  We have to enable this everwhere we had LNV running so this will mean the leafs, spines, and exit(routing) nodes.  The spines will learn the EVPN Type-2 and Type-3 routes from the leafs, and then distribute the routes to the other EVPN enabled neighbors.  It's only really two things. One, activate the l2vpn evpn address family for each neighbor. Then, enable advertise-all-vni.

Remeber, This won't take effect until we issue the 'net commit'

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible vtep -a 'net add bgp l2vpn evpn neighbor swp51-52 activate'
leaf02 | SUCCESS | rc=0 >>


exit01 | SUCCESS | rc=0 >>


leaf03 | SUCCESS | rc=0 >>


leaf01 | SUCCESS | rc=0 >>


leaf04 | SUCCESS | rc=0 >>


exit02 | SUCCESS | rc=0 >>


cumulus@oob-mgmt-server:~/lnv-to-evpn$
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible vtep -a 'net add bgp l2vpn evpn advertise-all-vni'
leaf02 | SUCCESS | rc=0 >>


exit01 | SUCCESS | rc=0 >>


leaf03 | SUCCESS | rc=0 >>


leaf01 | SUCCESS | rc=0 >>


leaf04 | SUCCESS | rc=0 >>


exit02 | SUCCESS | rc=0 >>
```

2. Then repeat those steps for the spines.  Ports 1-4 on the spines connect to leaf01-04.  Ports 29-30 connect to exit01 and exit02.

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible spine -a 'net add bgp l2vpn evpn neighbor swp1-4 activate'
spine01 | SUCCESS | rc=0 >>


spine02 | SUCCESS | rc=0 >>


cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible spine -a 'net add bgp l2vpn evpn neighbor swp29-30 activate'
spine02 | SUCCESS | rc=0 >>


spine01 | SUCCESS | rc=0 >>


cumulus@oob-mgmt-server:~/lnv-to-evpn$ 
```

3. Now lets stage the configuration change to disable bridge learning on all of the VTEP interfaces.

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible vtep -a 'net add vxlan vni-13 bridge learning off'
exit01 | SUCCESS | rc=0 >>


leaf01 | SUCCESS | rc=0 >>


leaf03 | SUCCESS | rc=0 >>


leaf02 | SUCCESS | rc=0 >>


leaf04 | SUCCESS | rc=0 >>


exit02 | SUCCESS | rc=0 >>


cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible vtep -a 'net add vxlan vni-24 bridge learning off'
leaf03 | SUCCESS | rc=0 >>


leaf01 | SUCCESS | rc=0 >>


exit01 | SUCCESS | rc=0 >>


leaf02 | SUCCESS | rc=0 >>


leaf04 | SUCCESS | rc=0 >>


exit02 | SUCCESS | rc=0 >>


cumulus@oob-mgmt-server:~/lnv-to-evpn$ 
```

4. Next we want to disable and stop the LNV service on all of the nodes where we have it enabled.  In our case, vxrd runs everywhere we have a VTEP (leafs and exit). Then the service nodes (vxsnd) is running on both spines.  We'll want to both stop it and also disable it so that it doesn't start again at reboot.

Note, we need --become for these commands

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible spine -a 'systemctl stop vxsnd.service' --become
spine01 | SUCCESS | rc=0 >>


spine02 | SUCCESS | rc=0 >>


cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible spine -a 'systemctl disable vxsnd.service' --become
spine01 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxsnd.service.

spine02 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxsnd.service.

cumulus@oob-mgmt-server:~/lnv-to-evpn$ 
```

Repeat these two steps on the VTEP nodes, but remember the service is vxrd here.

```
cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible vtep -a 'systemctl stop vxrd.service' --become
leaf01 | SUCCESS | rc=0 >>


leaf02 | SUCCESS | rc=0 >>


exit01 | SUCCESS | rc=0 >>


leaf04 | SUCCESS | rc=0 >>


leaf03 | SUCCESS | rc=0 >>


exit02 | SUCCESS | rc=0 >>


cumulus@oob-mgmt-server:~/lnv-to-evpn$ ansible vtep -a 'systemctl disable vxrd.service' --become
leaf01 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxrd.service.

exit01 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxrd.service.

leaf02 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxrd.service.

leaf03 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxrd.service.

leaf04 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxrd.service.

exit02 | SUCCESS | rc=0 >>
Removed symlink /etc/systemd/system/basic.target.wants/vxrd.service.

cumulus@oob-mgmt-server:~/lnv-to-evpn$ 
```

6. Commit the changes

7. Verify


---

>©2018 Cumulus Networks. CUMULUS, the Cumulus Logo, CUMULUS NETWORKS, and the Rocket Turtle Logo 
(the “Marks”) are trademarks and service marks of Cumulus Networks, Inc. in the U.S. and other 
countries. You are not permitted to use the Marks without the prior written consent of Cumulus 
Networks. The registered trademark Linux® is used pursuant to a sublicense from LMI, the exclusive 
licensee of Linus Torvalds, owner of the mark on a world-wide basis. All other marks are used under 
fair use or license from their respective owners.

For further details please see: [cumulusnetworks.com](http://www.cumulusnetworks.com)
