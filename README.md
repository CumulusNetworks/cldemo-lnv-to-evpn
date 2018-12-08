# Cumulus Linux LNV to EVPN Migration
![Reference Topology](./documentation/cldemo_topology.png "Reference Topology")

Welcome to the Cumulus Linux LNV to EVPN Migration demo . Using the Cumulus reference topology (embed link), this demo will walk through the steps to migrate your VXLAN control plane to the de facto standard using BGP EVPN.  LNV (the vxfld package) will be depricated in Cumulus Linux 4.x in favor of using EVPN to manage the VXLAN overlay. 

Link to the companion whitepaper:

Link to the Reference Topology:


### Provision the Topology and Log-in

    git clone https://github.com/cumulusnetworks/cldemo-lnv-to-evpn
    cd cldemo-lnv-to-evpn
    vagrant up oob-mgmt-server oob-mgmt-switch
    vagrant up
    vagrant ssh oob-mgmt-server
    *once logged into the oob-mgmt-server*
    ssh leaf01

---

>©2018 Cumulus Networks. CUMULUS, the Cumulus Logo, CUMULUS NETWORKS, and the Rocket Turtle Logo 
(the “Marks”) are trademarks and service marks of Cumulus Networks, Inc. in the U.S. and other 
countries. You are not permitted to use the Marks without the prior written consent of Cumulus 
Networks. The registered trademark Linux® is used pursuant to a sublicense from LMI, the exclusive 
licensee of Linus Torvalds, owner of the mark on a world-wide basis. All other marks are used under 
fair use or license from their respective owners.

For further details please see: [cumulusnetworks.com](http://www.cumulusnetworks.com)
