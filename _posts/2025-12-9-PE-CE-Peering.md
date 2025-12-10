---
title: "MPLS L3VPN: VRFs and PE-CE BGP Configuration"
date: 2025-12-09 00:00:00 -0500
categories: [MPLS, VPN]
tags: [mpls, vrf]
---

![MPLS Topology](/assets/img/MPLSLab1/Diagram.jpg)

## MPLS Lab: Log Entry #2

In the last post, we went over configuring OSPF(or any IGP) within the MPLS core and why it's important. If you missed it, to summarize:
- The configuration of the IGP is critical to MPLS because it advertises all networks within the core, providing end-to-end transit between Provider routers. 
- This gives LDP the ability to provide label bindings for IGP-learned routes, and those labels are used to make forwarding decisions. We will discuss that further in a subsequent post.

### VRFs

Before getting there, we need to create the L3VPN between the Provider Edge and the Customer Edge routers. To segregate traffic between customers, we must associate them to their own VRF(Virtual Routing and Forwarding) instance. 

Each VRF creates its own RIB and FIB, both of which are used to generate a unique routing and forwarding instance that is separate from the default global routing table. I will explain the topic of VRFs in more detail in a dedicated post. If you need a more in-depth explanation on routing tables and VRFs, please refer to the links below.
- <a href="https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cucme/vrf/design/guide/vrfDesignGuide.html" target="_blank">VRF Design Guide</a>
- <a href="https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5xx/routing/25xx/b-routing-cg-25xx-ncs540/overview-of-rib.html" target="_blank">RIB</a>

<br>


###### *PE-R1 VRF Config*

```
# Create VRFs

PE-R1(config)#
vrf definition CUSTOMER-A
 !
 address-family ipv4
 exit-address-family

vrf definition CUSTOMER-B
 !
 address-family ipv4
 exit-address-family

# Associate VRF to Interface
# Pull a backup of the config on the interface because the IPs will be reset after association.

PE-R1(config)#int g0/0
  vrf forwarding CUSTOMER-A
% Interface GigabitEthernet0/0 IPv4 disabled and address(es) removed due to enabling VRF CUSTOMER-A

PE-R1(config)#int g0/1
  vrf forwarding CUSTOMER-B
% Interface GigabitEthernet0/1 IPv4 disabled and address(es) removed due to enabling VRF CUSTOMER-B
```
<br>
<br>

###### *PE-R5 VRF Config*
```
# Create VRF

PE-R5(config)#vrf definition CUSTOMER-A
 address-family ipv4
 exit-address-family
vrf definition CUSTOMER-B
 address-family ipv4
 exit-address-family

# Associate VRF to Interface

PE-R5(config-vrf)#int g0/1
  vrf forwarding CUSTOMER-A
% Interface GigabitEthernet0/1 IPv4 disabled and address(es) removed due to enabling VRF CUSTOMER-A

PE-R5(config-if)#int g0/2
 vrf forwarding CUSTOMER-B
% Interface GigabitEthernet0/2 IPv4 disabled and address(es) removed due to enabling VRF CUSTOMER-B
```
<br>
<br>

##### *VRF Verification*
<br>

```
PE-R1#sh vrf br 
  Name                             Default RD            Protocols   Interfaces
  CUSTOMER-A                       <not set>             ipv4        Gi0/0
  CUSTOMER-B                       <not set>             ipv4        Gi0/1


PE-R1#sh ip route vrf CUSTOMER-A
Routing Table: CUSTOMER-A
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.10.0/30 is directly connected, GigabitEthernet0/0
L        10.0.10.2/32 is directly connected, GigabitEthernet0/0


PE-R1#sh ip route vrf CUSTOMER-B
Routing Table: CUSTOMER-B
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.100.0/30 is directly connected, GigabitEthernet0/1
L        10.0.100.2/32 is directly connected, GigabitEthernet0/1

-----

PE-R5#sh vrf br 
  Name                             Default RD            Protocols   Interfaces
  CUSTOMER-A                       <not set>             ipv4        Gi0/1
  CUSTOMER-B                       <not set>             ipv4        Gi0/2

PE-R5#sh ip route vrf CUSTOMER-A

Routing Table: CUSTOMER-A
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.10.4/30 is directly connected, GigabitEthernet0/1
L        10.0.10.5/32 is directly connected, GigabitEthernet0/1
PE-R5#sh ip route vrf CUSTOMER-B

Routing Table: CUSTOMER-B
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.100.4/30 is directly connected, GigabitEthernet0/2
L        10.0.100.5/32 is directly connected, GigabitEthernet0/2
```
<br>

### BGP

In the context of an MPLS L3VPN, BGP is used between the PE and CE routers to advertise customer LAN networks into the VRF. BGP's role in this scenario is primarily to serve as the control-plane protocol that delivers customer routes to the MPLS backbone, where the Label Switching Routers(LSRs) can assign labels and provide end-to-end reachability between sites. Note that the BGP syntax on the PE routers will vary slightly due to the introduction of VRFs. Please see the configuration below for reference.

<br>
###### *Customer-Routers*
```
CE-R1(config-router-af)#do sh run | sec bgp
router bgp 100
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 10.0.10.2 remote-as 300
 !
 address-family ipv4
  network 192.168.1.0
  neighbor 10.0.10.2 activate
 exit-address-family

CE-R2(config-router-af)#do sh run | sec bgp
router bgp 100
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 10.0.10.5 remote-as 300
 !
 address-family ipv4
  network 192.168.2.0
  neighbor 10.0.10.5 activate
 exit-address-family

CE-RA(config-router-af)#do sh run | sec bgp
router bgp 200
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 10.0.100.2 remote-as 300
 !
 address-family ipv4
  network 192.168.1.0
  neighbor 10.0.100.2 activate
 exit-address-family

CE-RB(config-router-af)#do sh run | sec bgp
router bgp 200
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 10.0.100.5 remote-as 300
 !
 address-family ipv4
  network 192.168.2.0
  neighbor 10.0.100.5 activate
 exit-address-family
```

When we attempt to configure BGP peering for the PE routers, we get the following error `% VRF CUSTOMER-A does not have an RD configured.`

***BGP requires VRF instances to have a Route Distinguisher(RD) to differentiate between potential overlapping address spaces.*** Below is an example of the creation of RDs within the VRFs, VRF BGP peering, and the RD value prepended to a learned network prefix.

<br>
<br>
###### *PE-R1*

```
PE-R1#sh run vrf CUSTOMER-A
vrf definition CUSTOMER-A
 rd 300:1
 !
 address-family ipv4
 exit-address-family
!

PE-R1#sh run vrf CUSTOMER-B
vrf definition CUSTOMER-B
 rd 300:2
 !
 address-family ipv4
 exit-address-family
!

PE-R1(config-vrf)#router bgp 300
PE-R1(config-router)#address-family ipv4 unicast vrf CUSTOMER-A
PE-R1(config-router-af)#neighbor 10.0.10.1 remote-as 100
*Dec  4 03:02:57.895: %BGP-5-ADJCHANGE: neighbor 10.0.10.1 vpn vrf CUSTOMER-A Up


PE-R1(config-router-af)#address-family ipv4 vrf CUSTOMER-B
PE-R1(config-router-af)#neighbor 10.0.100.1 remote-as 200
*Dec  4 03:04:45.662: %BGP-5-ADJCHANGE: neighbor 10.0.100.1 vpn vrf CUSTOMER-B Up 
```
<br>
<br>
###### *PE-R5*

```
PE-R5(config-vrf)#do sh run vrf CUSTOMER-A
vrf definition CUSTOMER-A
 rd 300:1
 !
 address-family ipv4
 exit-address-family
!

PE-R5(config-vrf)#do sh run vrf CUSTOMER-B
vrf definition CUSTOMER-B
 rd 300:2
 !
 address-family ipv4
 exit-address-family
!

PE-R5(config-vrf)#router bgp 300
PE-R5(config-router)#address-family ipv4 vrf CUSTOMER-A
PE-R5(config-router-af)#neighbor 10.0.10.6 remote-as 100
*Dec  4 03:46:43.375: %BGP-5-ADJCHANGE: neighbor 10.0.10.6 vpn vrf CUSTOMER-A Up

PE-R5(config-router-af)#address-family ipv4 vrf CUSTOMER-B
PE-R5(config-router-af)#neighbor 10.0.100.6 remote-as 200
*Dec  4 03:47:53.965: %BGP-5-ADJCHANGE: neighbor 10.0.100.6 vpn vrf CUSTOMER-B Up 
```
<br>
<br>

##### *BGP Verification*

<br>
###### *PE-R1*

```
PE-R1#sh vrf br
  Name                             Default RD            Protocols   Interfaces
  CUSTOMER-A                       300:1                 ipv4        Gi0/0
  CUSTOMER-B                       300:2                 ipv4        Gi0/1

PE-R1#sh bgp vpnv4 unicast vrf CUSTOMER-A summ
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.10.1       4          100      15      15        3    0    0 00:10:04        1

PE-R1#sh bgp vpnv4 unicast vrf CUSTOMER-B summ
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.100.1      4          200      14      13        3    0    0 00:08:43        1

PE-R1#sh bgp vpnv4 unicast vrf CUSTOMER-A 192.168.1.0/24
BGP routing table entry for 300:1:192.168.1.0/24, version 2
Paths: (1 available, best #1, table CUSTOMER-A)
  Not advertised to any peer
  Refresh Epoch 1
  100
    10.0.10.1 (via vrf CUSTOMER-A) from 10.0.10.1 (192.168.1.1)
      Origin IGP, metric 0, localpref 100, valid, external, best
      rx pathid: 0, tx pathid: 0x0

PE-R1#sh bgp vpnv4 unicast vrf CUSTOMER-B 192.168.1.0/24
BGP routing table entry for 300:2:192.168.1.0/24, version 3
Paths: (1 available, best #1, table CUSTOMER-B)
  Not advertised to any peer
  Refresh Epoch 1
  200
    10.0.100.1 (via vrf CUSTOMER-B) from 10.0.100.1 (192.168.1.1)
      Origin IGP, metric 0, localpref 100, valid, external, best
      rx pathid: 0, tx pathid: 0x0

PE-R1#sh bgp vpnv4 unicast vrf CUSTOMER-A
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>   192.168.1.0      10.0.10.1                0             0 100 i

PE-R1#sh bgp vpnv4 unicast vrf CUSTOMER-B
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
```
<br>
###### *PE-R5*

```
PE-R5#sh vrf br
  Name                             Default RD            Protocols   Interfaces
  CUSTOMER-A                       300:1                 ipv4        Gi0/1
  CUSTOMER-B                       300:2                 ipv4        Gi0/2

PE-R5#sh bgp vpnv4 unicast vrf CUSTOMER-A summ
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.10.6       4          100       7       6        3    0    0 00:02:27        1


PE-R5#sh bgp vpnv4 unicast vrf CUSTOMER-B summ
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.100.6      4          200       6       5        3    0    0 00:01:31        1

PE-R5#sh bgp vpnv4 unicast vrf CUSTOMER-A 192.168.2.0/24
BGP routing table entry for 300:1:192.168.2.0/24, version 2
Paths: (1 available, best #1, table CUSTOMER-A)
  Not advertised to any peer
  Refresh Epoch 1
  100
    10.0.10.6 (via vrf CUSTOMER-A) from 10.0.10.6 (192.168.2.1)
      Origin IGP, metric 0, localpref 100, valid, external, best
      rx pathid: 0, tx pathid: 0x0

PE-R5#sh bgp vpnv4 unicast vrf CUSTOMER-B 192.168.2.0/24
BGP routing table entry for 300:2:192.168.2.0/24, version 3
Paths: (1 available, best #1, table CUSTOMER-B)
  Not advertised to any peer
  Refresh Epoch 1
  200
    10.0.100.6 (via vrf CUSTOMER-B) from 10.0.100.6 (192.168.2.1)
      Origin IGP, metric 0, localpref 100, valid, external, best
      rx pathid: 0, tx pathid: 0x0

PE-R5#sh bgp vpnv4 unicast vrf CUSTOMER-A
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>   192.168.2.0      10.0.10.6                0             0 100 i

PE-R5#sh bgp vpnv4 unicast vrf CUSTOMER-B
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
 *>   192.168.2.0      10.0.100.6               0             0 200 i
 ```

<br>

### Next Steps

Now that the PEs have learned the LAN networks of their respective client sites, they must now share that information with each other to achieve end-to-end connectivity between customers. This can be accomplished through MP-BGP peering, which we will discuss in the next post.

Thanks for reading.