---
title: "MPLS L3VPN: PE-PE MP-BGP Peering"
date: 2025-12-18 00:00:00 -0500
categories: [MPLS, VPN]
tags: [mpls, vrf]
---

![MPLS Topology](/assets/img/MPLSLab1/MPLS.jpeg)
<a href="https://github.com/kadeeeem/ENARSI/tree/main/MPLS" target="_blank">Lab Files</a>

## MPLS Lab: Log Entry #3

In our previous entry, we created the unqiue customer VRFs, introduced route distinguishers, established PE-CE BGP peering, and confirmed that the LAN networks of the customer were advertised to each PE router.

Now, our objective is to share the learned routes between each PE router. This will be accomplished through the use of <a href="https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/113555-mp-ebgp-config-00.html" target="_blank">***Multiprotocol-BGP(MP-BGP)***</a>.

<br>

### MP-BGP Overview

So what makes MP-BGP different from BGP?


When BGP was standardized as **BGP-4 (RFC 1771)**, it primarily operated using the IPv4 address family. Over the years, other **address family indicators(AFIs)** and **subsquent address family indicators(SAFIs)** were standardized and introduced to the original protocol, leading to the term "MP-BGP". Technically, this makes all modern implementations of BGP = MP-BGP. 

Though, when speaking amongst engineers and when referencing Cisco documentation, MP-BGP is the term used to differeniate between designs that use VPNv4/VPNv6, EVPN, etc. 
These address families behave differently when compared to conventional IPv4/IPv6 because they carry:
- Route Distinguishers(RDs)
- Route Targets(RTs)
- RIBs separate from the global routing table

As seen in the previous post, we can confirm this behavior from the slight variation in configuration syntax when forming the adjacency with a neighbor(Customer Routers) that is on a link associated to a VRF.

<br>

### Implementing MP-BGP between Provider Edge Routers

MP-BGP between the PE routers is required for the sharing of learned routes from CE routers. Below are the configuration snippets used to form the adjacency between each PE router, along with the initialization of VPNv4 capabilities.

```
# MP-BGP Peering

PE-R1
router bgp 300
 neighbor 5.5.5.5 remote-as 300
 neighbor 5.5.5.5 update-source Loopback0
 !
 address-family ipv4
  neighbor 5.5.5.5 activate
 exit-address-family
!
address-family vpnv4
  neighbor 5.5.5.5 activate
  neighbor 5.5.5.5 send-community extended


PE-R5
router bgp 300
 bgp log-neighbor-changes
 neighbor 1.1.1.1 remote-as 300
 neighbor 1.1.1.1 update-source Loopback0
 !
 address-family ipv4
  neighbor 1.1.1.1 activate
 exit-address-family
 !
address-family vpnv4
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community extended
 exit-address-family
```

```
# MP-BGP Peering Verification

PE-R1#show bgp ipv4 unicast neighbors 
BGP neighbor is 5.5.5.5,  remote AS 300, internal link
  BGP version 4, remote router ID 5.5.5.5
  BGP state = Established, up for 00:02:48
  Neighbor sessions:
    1 active

PE-R5#show bgp ipv4 unicast neighbors 
BGP neighbor is 1.1.1.1,  remote AS 300, internal link
  BGP version 4, remote router ID 1.1.1.1
  BGP state = Established, up for 00:03:30
  Neighbor sessions:
    1 active
```
<br>

Now that the PE-PE adjacency has been established, we need to advertise the learned routes from the CE routers in each VRF and share them with the remote PE node.  Let's confirm that we know the routes by running some show commands:

```
PE-R1#sh bgp vpnv4 unicast all

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>   192.168.1.0      10.0.10.1                0             0 100 i
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
 *>   192.168.1.0      10.0.100.1               0             0 200 i


PE-R5#sh bgp vpnv4 unicast all 

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>   192.168.2.0      10.0.10.6                0             0 100 i
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
 *>   192.168.2.0      10.0.100.6               0             0 200 i
```

<br>

With the learned routes verified, lets advertise them to the remote PE node through the VRF address-family.

```
PE-R1
address-family ipv4 vrf CUSTOMER-A
  network 192.168.1.0
  neighbor 10.0.10.1 remote-as 100
  neighbor 10.0.10.1 activate
 exit-address-family
 !
 address-family ipv4 vrf CUSTOMER-B
  network 192.168.1.0
  neighbor 10.0.100.1 remote-as 200
  neighbor 10.0.100.1 activate
```
<br>

If you're following along, you'll notice that even after advertising the routes, they still do not show in the routing table of the respective VRFs on PE-R5. And we know for a fact that PE-R1 is sending these routes because of the verifcation command `show bgp vpnv4 unicast all neighbors 5.5.5.5 advertised-routes`.

```
PE-R1#sh bgp vpnv4 unicast all neighbors 5.5.5.5 advertised-routes 

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>   192.168.1.0      10.0.10.1                0             0 100 i
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
 *>   192.168.1.0      10.0.100.1               0             0 200 i


PE-R5#sh bgp vpnv4 unicast all 
BGP table version is 3, local router ID is 5.5.5.5
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>   192.168.2.0      10.0.10.6                0             0 100 i
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
 *>   192.168.2.0      10.0.100.6               0             0 200 i
```

The reason we are unable to advertise the routes successfully is because of a critical missing component that is crucial to MP-BGP VRFs, ***Route Targets(RT)***.

RTs, in this scenario, are used to import and export customer routes across VRF instances. The RT, like the RD, is configured under the customer VRF instance in ***two octect ASN-based format(ASN:Value)***.

<br>

RTs have two values:
- Import-value — accepts routes that are prepended with this value
- Export-value — prepends routes that are being advertised with this value
With that said, the PE nodes must have their import and export values configured inversely to successfully advertise/learn routes if these values are different. For simplicity's sake, we will make both values the same in this scenario.

Below are the commands used to configure RTs on the respective routers:

```
PE-R1
vrf definition CUSTOMER-A
  route-target both 300:100
vrf def CUSTOMER-B
  route-target both 300:200

PE-R1#sh vrf detail 
VRF CUSTOMER-A (VRF Id = 1); default RD 300:1; default VPNID <not set>
   Export VPN route-target communities
    RT:300:100              
  Import VPN route-target communities
    RT:300:100              
  
VRF CUSTOMER-B (VRF Id = 2); default RD 300:2; default VPNID <not set>
    Export VPN route-target communities
    RT:300:200              
  Import VPN route-target communities
    RT:300:200              


PE-R5
vrf definition CUSTOMER-A
  route-target both 300:100
vrf def CUSTOMER-B
  route-target both 300:200


PE-R5#sh vrf detail 
VRF CUSTOMER-A (VRF Id = 1); default RD 300:1; default VPNID <not set>
  Export VPN route-target communities
    RT:300:100              
  Import VPN route-target communities
    RT:300:100              

VRF CUSTOMER-B (VRF Id = 2); default RD 300:2; default VPNID <not set>
  Export VPN route-target communities
    RT:300:200              
  Import VPN route-target communities
    RT:300:200     
```

<br>

Finally, when we run our verification commands, we now see that the PE nodes have learned the advertised networks from the remote CE devices.

```
PE-R1#sh bgp vpnv4 unicast all

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>   192.168.1.0      10.0.10.1                0             0 100 i
 *>i  192.168.2.0      5.5.5.5                  0    100      0 100 i
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
 *>   192.168.1.0      10.0.100.1               0             0 200 i
 *>i  192.168.2.0      5.5.5.5                  0    100      0 200 i


PE-R5#sh bgp vpnv4 unicast all

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 300:1 (default for vrf CUSTOMER-A)
 *>i  192.168.1.0      1.1.1.1                  0    100      0 100 i
 *>   192.168.2.0      10.0.10.6                0             0 100 i
Route Distinguisher: 300:2 (default for vrf CUSTOMER-B)
 *>i  192.168.1.0      1.1.1.1                  0    100      0 200 i
 *>   192.168.2.0      10.0.100.6               0             0 200 i
```

After testing pings, we can conclude that end-to-end reachability still has not been established. This is because the intermediate Provider routers are not participating in BGP and have no awareness of the customer VRFs. This is where MPLS comes into play. I'll go over the theory and configuration steps in the next post.

Thanks for reading.
