Purpose
=======

Spin up vagrant multi node routing environment and manage all nodes using Ansible.

Follow the "Hey I can DevOPS my Network too!" series on my blog at the following
link. http://everythingshouldbevirtual.com/hey-i-can-devops-my-network-too-intro
* Note: The above blog post is out of date until it is updated based on the new
configuration here.

Requirements
------------

* [Ansible]
* [VirtualBox]
* [Vagrant]

Variable Definitions
--------------------
`group_vars/quagga-routers/quagga.yml`
```
---
quagga_bgp_router_configs:
  - name: 'node0'
    local_as: '64512'
    neighbors:
      - neighbor: '192.168.1.11'
        description: 'node1'
        remote_as: '64512'
        prefix_lists: # Define specific BGP neighbore prefix lists
          - name: 'FILTER01-out' # Define name of filter
            direction: 'out' # define direction (in|out)
            orf: 'send' # Define outbound route filter (send|receive|both)
      - neighbor: '192.168.2.12'
        description: 'node2'
        prefix_lists:
          - name: 'FILTER02-in'
            direction: 'in'
            orf: 'receive'
        remote_as: '64513'
      - neighbor: '192.168.3.13'
        description: 'node3'
        remote_as: '64514'
    network_advertisements:  #networks to advertise
      - '10.0.0.10/32'
      - '192.168.1.0/24'
      - '192.168.2.0/24'
      - '192.168.3.0/24'
    router_id: '10.0.0.10'
    prefix_lists:
      - name: 'FILTER01-out'
        action: 'permit'
        network: '10.0.0.10/32'
        sequence: '10'
      - name: 'FILTER01-out'
        action: 'permit'
        network: '192.168.1.0/24'
        sequence: '20'
      - name: 'FILTER01-out'
        action: 'permit'
        network: '192.168.2.0/24'
        sequence: '30'
      - name: 'FILTER01-out'
        action: 'deny'
        network: 'any'
        sequence: '40'
  - name: 'node1'
    local_as: '64512'
    neighbors:
      - neighbor: '192.168.1.10' # Peering with loopback address for iBGP
        description: 'node0'
        remote_as: '64512'
        prefix_lists:
          - name: 'FILTER01-in'
            direction: 'in'
            orf: 'receive'
    network_advertisements:  #networks to advertise
      - '10.1.1.11/32'
      - '192.168.1.0/24'
    router_id: '10.1.1.11'
  - name: 'node2'
    local_as: '64513'
    neighbors:
      - neighbor: '192.168.2.10'
        description: 'node0'
        prefix_lists:
          - name: 'FILTER02-out'
            direction: 'out'
            orf: 'send'
        remote_as: '64512'
    network_advertisements:  #networks to advertise
      # - '10.2.2.12/32'
      - '192.168.2.0/24'
    prefix_lists:
      - name: 'FILTER02-out'
        action: 'permit'
        network: '192.168.2.0/24'
        sequence: '10'
      - name: 'FILTER01-out'
        action: 'deny'
        network: 'any'
        sequence: '20'
    router_id: '10.2.2.12'
  - name: 'node3'
    local_as: '64514'
    neighbors:
      - neighbor: '192.168.3.10'
        description: 'node0'
        remote_as: '64512'
    network_advertisements:  #networks to advertise
      - '10.3.3.13/32'
      - '192.168.3.0/24'
    router_id: '10.3.3.13'
quagga_config_bgpd: false #defines if quagga bgpd should be configured based on quagga_bgp_router_configs...makes it easy to disable auto routing in order to define your routes manually
quagga_config_interfaces: true
quagga_config_ospfd: false  #defines if quagga ospfd should be configured based on quagga_ospf_ vars...makes it easy to disable auto routing in order to define your routes manually
quagga_config: true
quagga_enable_bgpd: false
quagga_enable_ospfd: false
```
`host_vars/node0/quagga.yml`
```
---
quagga_interfaces_lo:
  - int: "lo{{ ':' }}0"
    address: '10.0.0.10/32'
    method: 'static'
    configure: true
  # - int: "lo{{ ':' }}1"
  #   address: '10.0.0.11/32'
  #   method: 'static'
  #   configure: false
  # - int: "lo{{ ':' }}2"
  #   address: '10.0.0.12/32'
  #   method: 'static'
  #   configure: false
quagga_ospf_area_config:
  - network: '192.168.1.0/24'
    area: '{{ quagga_ospf_area }}'
  - network: '192.168.2.0/24'
    area: '{{ quagga_ospf_area }}'
  - network: '192.168.3.0/24'
    area: '{{ quagga_ospf_area }}'
quagga_ospf_routerid: '10.0.0.10'

```
`host_vars/node1/quagga.yml`
```
---
quagga_interfaces_lo:
  - int: "lo{{ ':' }}0"
    address: '10.1.1.11/32'
    method: 'static'
    configure: true
quagga_ospf_area_config:
  - network: '192.168.1.0/24'
    area: '{{ quagga_ospf_area }}'
quagga_ospf_routerid: '10.1.1.11'
```
`host_vars/node2/quagga.yml`
```
---
quagga_interfaces_lo:
  - int: "lo{{ ':' }}0"
    address: '10.2.2.12/32'
    method: 'static'
    configure: true
quagga_ospf_area_config:
  - network: '192.168.2.0/24'
    area: '{{ quagga_ospf_area }}'
quagga_ospf_routerid: '10.2.2.12'
```
`host_vars/node3/quagga.yml`
```
---
quagga_interfaces_lo:
  - int: "lo{{ ':' }}0"
    address: '10.3.3.13/32'
    method: 'static'
    configure: true
quagga_ospf_area_config:
  - network: '192.168.3.0/24'
    area: '{{ quagga_ospf_area }}'
quagga_ospf_routerid: '10.3.3.13'
```

Usage
-----

```
git clone https://github.com/mrlesmithjr/vagrant-ansible-routing-template.git
cd vagrant-ansible-routing-template
```

Spin up your environment
```
vagrant up
```

##### OSPF

If you define `quagga_enable_ospfd: true`, `quagga_config_ospfd: true`,
`quagga_config_bgpd: false`, and `quagga_enable_bgpd: false` in
`group_vars/quagga-routers/quagga.yml` and then run
`ansible-playbook -i hosts playbook.yml` to configure OSPF you can then do some
validation by running the following commands:
```
vagrant ssh node0
```
Validate your IP routes:
```
vagrant@node0:~$ ip route
default via 10.0.2.2 dev enp0s3
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15
10.1.1.11 via 192.168.1.11 dev enp0s9  proto zebra  metric 20
10.2.2.12 via 192.168.2.12 dev enp0s10  proto zebra  metric 20
10.3.3.13 via 192.168.3.13 dev enp0s16  proto zebra  metric 20
192.168.1.0/24 dev enp0s9  proto kernel  scope link  src 192.168.1.10
192.168.2.0/24 dev enp0s10  proto kernel  scope link  src 192.168.2.10
192.168.3.0/24 dev enp0s16  proto kernel  scope link  src 192.168.3.10
192.168.250.0/24 dev enp0s8  proto kernel  scope link  src 192.168.250.10
```
Now telnet to the ospfd daemon and run some additional validations:
```
telnet localhost 2604
```
Enter `quagga` for the password:
```
node0> sh ip ospf neighbor

    Neighbor ID Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
10.1.1.11         1 Full/DR           31.979s 192.168.1.11    enp0s9:192.168.1.10      0     0     0
10.2.2.12         1 Full/DR           31.972s 192.168.2.12    enp0s10:192.168.2.10     0     0     0
10.3.3.13         1 Full/DR           31.996s 192.168.3.13    enp0s16:192.168.3.10     0     0     0
```
Additional sh ip ospf commands.
```
node0> sh ip ospf border-routers
============ OSPF router routing table =============
R    10.1.1.11             [10] area: 0.0.0.51, ASBR
                           via 192.168.1.11, enp0s9
R    10.2.2.12             [10] area: 0.0.0.51, ASBR
                           via 192.168.2.12, enp0s10
R    10.3.3.13             [10] area: 0.0.0.51, ASBR
                           via 192.168.3.13, enp0s16
```
```
node0> sh ip ospf database

       OSPF Router with ID (10.0.0.10)

                Router Link States (Area 0.0.0.51)

Link ID         ADV Router      Age  Seq#       CkSum  Link count
10.0.0.10       10.0.0.10        366 0x8000000b 0xd547 3
10.1.1.11       10.1.1.11        366 0x80000004 0x9695 1
10.2.2.12       10.2.2.12        371 0x80000004 0x8e93 1
10.3.3.13       10.3.3.13        371 0x80000004 0x8691 1

                Net Link States (Area 0.0.0.51)

Link ID         ADV Router      Age  Seq#       CkSum
192.168.1.11    10.1.1.11        367 0x80000001 0x772b
192.168.2.12    10.2.2.12        372 0x80000001 0x7426
192.168.3.13    10.3.3.13        372 0x80000001 0x7121

                AS External Link States

Link ID         ADV Router      Age  Seq#       CkSum  Route
10.0.0.10       10.0.0.10        366 0x80000005 0x266c E2 10.0.0.10/32 [0x0]
10.0.2.0        10.0.0.10        366 0x80000005 0x7426 E2 10.0.2.0/24 [0x0]
10.0.2.0        10.1.1.11        366 0x80000002 0x6535 E2 10.0.2.0/24 [0x0]
10.0.2.0        10.2.2.12        371 0x80000002 0x5047 E2 10.0.2.0/24 [0x0]
10.0.2.0        10.3.3.13        371 0x80000002 0x3b59 E2 10.0.2.0/24 [0x0]
10.1.1.11       10.1.1.11        366 0x80000002 0xf599 E2 10.1.1.11/32 [0x0]
10.2.2.12       10.2.2.12        371 0x80000002 0xbfc9 E2 10.2.2.12/32 [0x0]
10.3.3.13       10.3.3.13        371 0x80000002 0x89f9 E2 10.3.3.13/32 [0x0]
192.168.250.0   10.0.0.10        366 0x80000005 0x92af E2 192.168.250.0/24 [0x0]
192.168.250.0   10.1.1.11        366 0x80000003 0x81bf E2 192.168.250.0/24 [0x0]
192.168.250.0   10.2.2.12        371 0x80000003 0x6cd1 E2 192.168.250.0/24 [0x0]
192.168.250.0   10.3.3.13        371 0x80000003 0x57e3 E2 192.168.250.0/24 [0x0]
```
```
node0> sh ip ospf interface
enp0s3 is up
  ifindex 2, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
enp0s8 is up
  ifindex 3, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
enp0s9 is up
  ifindex 4, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 192.168.1.10/24, Broadcast 192.168.1.255, Area 0.0.0.51
  MTU mismatch detection:enabled
  Router ID 10.0.0.10, Network Type BROADCAST, Cost: 10
  Transmit Delay is 1 sec, State Backup, Priority 1
  Designated Router (ID) 10.1.1.11, Interface Address 192.168.1.11
  Backup Designated Router (ID) 10.0.0.10, Interface Address 192.168.1.10
  Multicast group memberships: OSPFAllRouters OSPFDesignatedRouters
  Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
    Hello due in 4.123s
  Neighbor Count is 1, Adjacent neighbor count is 1
enp0s10 is up
  ifindex 5, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 192.168.2.10/24, Broadcast 192.168.2.255, Area 0.0.0.51
  MTU mismatch detection:enabled
  Router ID 10.0.0.10, Network Type BROADCAST, Cost: 10
  Transmit Delay is 1 sec, State Backup, Priority 1
  Designated Router (ID) 10.2.2.12, Interface Address 192.168.2.12
  Backup Designated Router (ID) 10.0.0.10, Interface Address 192.168.2.10
  Multicast group memberships: OSPFAllRouters OSPFDesignatedRouters
  Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
    Hello due in 4.123s
  Neighbor Count is 1, Adjacent neighbor count is 1
enp0s16 is up
  ifindex 6, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 192.168.3.10/24, Broadcast 192.168.3.255, Area 0.0.0.51
  MTU mismatch detection:enabled
  Router ID 10.0.0.10, Network Type BROADCAST, Cost: 10
  Transmit Delay is 1 sec, State Backup, Priority 1
  Designated Router (ID) 10.3.3.13, Interface Address 192.168.3.13
  Backup Designated Router (ID) 10.0.0.10, Interface Address 192.168.3.10
  Multicast group memberships: OSPFAllRouters OSPFDesignatedRouters
  Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
    Hello due in 4.123s
  Neighbor Count is 1, Adjacent neighbor count is 1
lo is up
  ifindex 1, MTU 65536 bytes, BW 0 Kbit <UP,LOOPBACK,RUNNING>
  OSPF not enabled on this interface
```
```
node0> sh ip ospf route
============ OSPF network routing table ============
N    192.168.1.0/24        [10] area: 0.0.0.51
                           directly attached to enp0s9
N    192.168.2.0/24        [10] area: 0.0.0.51
                           directly attached to enp0s10
N    192.168.3.0/24        [10] area: 0.0.0.51
                           directly attached to enp0s16

============ OSPF router routing table =============
R    10.1.1.11             [10] area: 0.0.0.51, ASBR
                           via 192.168.1.11, enp0s9
R    10.2.2.12             [10] area: 0.0.0.51, ASBR
                           via 192.168.2.12, enp0s10
R    10.3.3.13             [10] area: 0.0.0.51, ASBR
                           via 192.168.3.13, enp0s16

============ OSPF external routing table ===========
N E2 10.0.2.0/24           [10/20] tag: 0
                           via 192.168.1.11, enp0s9
                           via 192.168.2.12, enp0s10
                           via 192.168.3.13, enp0s16
N E2 10.1.1.11/32          [10/20] tag: 0
                           via 192.168.1.11, enp0s9
N E2 10.2.2.12/32          [10/20] tag: 0
                           via 192.168.2.12, enp0s10
N E2 10.3.3.13/32          [10/20] tag: 0
                           via 192.168.3.13, enp0s16
N E2 192.168.250.0/24      [10/20] tag: 0
                           via 192.168.1.11, enp0s9
                           via 192.168.2.12, enp0s10
                           via 192.168.3.13, enp0s16
```

##### BGP

If you define `quagga_enable_ospfd: false`, `quagga_config_ospfd: false`,
`quagga_config_bgpd: true`, and `quagga_enable_bgpd: true` in
`group_vars/quagga-routers/quagga.yml` and then run
`ansible-playbook -i hosts playbook.yml` to configure BGP you can then do some
validation by running the following commands:
```
vagrant ssh node0
```
```
vagrant@node0:~$ ip route
default via 10.0.2.2 dev enp0s3
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15
10.1.1.11 via 192.168.1.11 dev enp0s9  proto zebra
10.3.3.13 via 192.168.3.13 dev enp0s16  proto zebra
192.168.1.0/24 dev enp0s9  proto kernel  scope link  src 192.168.1.10
192.168.2.0/24 dev enp0s10  proto kernel  scope link  src 192.168.2.10
192.168.3.0/24 dev enp0s16  proto kernel  scope link  src 192.168.3.10
192.168.250.0/24 dev enp0s8  proto kernel  scope link  src 192.168.250.10
```
Now telnet to the bgpd daemon and run some additional validations:
```
telnet localhost 2605
```
Enter `quagga` for the password:
```
node0> sh ip bgp
BGP table version is 0, local router ID is 10.0.0.10
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.0.0.10/32     0.0.0.0                  0         32768 i
*>i10.1.1.11/32     192.168.1.11             0    100      0 i
*> 10.3.3.13/32     192.168.3.13             0             0 64514 i
* i192.168.1.0      192.168.1.11             0    100      0 i
*>                  0.0.0.0                  0         32768 i
*  192.168.2.0      192.168.2.12             0             0 64513 i
*>                  0.0.0.0                  0         32768 i
*  192.168.3.0      192.168.3.13             0             0 64514 i
*>                  0.0.0.0                  0         32768 i

Total number of prefixes 6
```
```
node0> sh ip bgp neighbors
BGP neighbor is 192.168.1.11, remote AS 64512, local AS 64512, internal link
 Description: "node1"
  BGP version 4, remote router ID 10.1.1.11
  BGP state = Established, up for 00:19:21
  Last read 00:00:21, hold time is 180, keepalive interval is 60 seconds
  Neighbor capabilities:
    4 Byte AS: advertised and received
    Route refresh: advertised and received(old & new)
    Address family IPv4 Unicast: advertised and received
    Graceful Restart Capabilty: advertised and received
      Remote Restart timer is 120 seconds
      Address families by peer:
        none
  Graceful restart informations:
    End-of-RIB send: IPv4 Unicast
    End-of-RIB received: IPv4 Unicast
  Message statistics:
    Inq depth is 0
    Outq depth is 0
                         Sent       Rcvd
    Opens:                  2          0
    Notifications:          0          0
    Updates:                2          2
    Keepalives:            21         20
    Route Refresh:          1          0
    Capability:             0          0
    Total:                 26         22
  Minimum time between advertisement runs is 5 seconds

 For address family: IPv4 Unicast
  AF-dependant capabilities:
    Outbound Route Filter (ORF) type (64) Prefix-list:
      Send-mode: advertised
      Receive-mode: received
    Outbound Route Filter (ORF) type (128) Prefix-list:
      Send-mode: advertised
      Receive-mode: received
  Inbound soft reconfiguration allowed
  NEXT_HOP is always this router
  Community attribute sent to this neighbor(both)
  Outbound path policy configured
  Outgoing update prefix filter list is *FILTER01-out
  2 accepted prefixes

  Connections established 1; dropped 0
  Last reset never
Local host: 192.168.1.10, Local port: 179
Foreign host: 192.168.1.11, Foreign port: 16080
Nexthop: 192.168.1.10
Nexthop global: fe80::a00:27ff:fecc:87df
Nexthop local: ::
BGP connection: non shared network
Read thread: on  Write thread: off

BGP neighbor is 192.168.2.12, remote AS 64513, local AS 64512, external link
 Description: "node2"
  BGP version 4, remote router ID 10.2.2.12
  BGP state = Established, up for 00:19:17
  Last read 00:00:17, hold time is 180, keepalive interval is 60 seconds
  Neighbor capabilities:
    4 Byte AS: advertised and received
    Route refresh: advertised and received(old & new)
    Address family IPv4 Unicast: advertised and received
    Graceful Restart Capabilty: advertised and received
      Remote Restart timer is 120 seconds
      Address families by peer:
        none
  Graceful restart informations:
    End-of-RIB send: IPv4 Unicast
    End-of-RIB received: IPv4 Unicast
  Message statistics:
    Inq depth is 0
    Outq depth is 0
                         Sent       Rcvd
    Opens:                  1          1
    Notifications:          0          0
    Updates:                4          2
    Keepalives:            21         20
    Route Refresh:          0          1
    Capability:             0          0
    Total:                 26         24
  Minimum time between advertisement runs is 30 seconds

 For address family: IPv4 Unicast
  AF-dependant capabilities:
    Outbound Route Filter (ORF) type (64) Prefix-list:
      Send-mode: received
      Receive-mode: advertised
    Outbound Route Filter (ORF) type (128) Prefix-list:
      Send-mode: received
      Receive-mode: advertised
  Inbound soft reconfiguration allowed
  NEXT_HOP is always this router
  Community attribute sent to this neighbor(both)
  1 accepted prefixes

  Connections established 1; dropped 0
  Last reset never
Local host: 192.168.2.10, Local port: 19768
Foreign host: 192.168.2.12, Foreign port: 179
Nexthop: 192.168.2.10
Nexthop global: fe80::a00:27ff:fe6a:39a9
Nexthop local: ::
BGP connection: non shared network
Read thread: on  Write thread: off

BGP neighbor is 192.168.3.13, remote AS 64514, local AS 64512, external link
 Description: "node3"
  BGP version 4, remote router ID 10.3.3.13
  BGP state = Established, up for 00:19:21
  Last read 00:00:21, hold time is 180, keepalive interval is 60 seconds
  Neighbor capabilities:
    4 Byte AS: advertised and received
    Route refresh: advertised and received(old & new)
    Address family IPv4 Unicast: advertised and received
    Graceful Restart Capabilty: advertised and received
      Remote Restart timer is 120 seconds
      Address families by peer:
        none
  Graceful restart informations:
    End-of-RIB send: IPv4 Unicast
    End-of-RIB received: IPv4 Unicast
  Message statistics:
    Inq depth is 0
    Outq depth is 0
                         Sent       Rcvd
    Opens:                  2          0
    Notifications:          0          0
    Updates:                3          2
    Keepalives:            21         20
    Route Refresh:          0          0
    Capability:             0          0
    Total:                 26         22
  Minimum time between advertisement runs is 30 seconds

 For address family: IPv4 Unicast
  Inbound soft reconfiguration allowed
  NEXT_HOP is always this router
  Community attribute sent to this neighbor(both)
  2 accepted prefixes

  Connections established 1; dropped 0
  Last reset never
Local host: 192.168.3.10, Local port: 179
Foreign host: 192.168.3.13, Foreign port: 44638
Nexthop: 192.168.3.10
Nexthop global: fe80::a00:27ff:fe34:abaa
Nexthop local: ::
BGP connection: non shared network
Read thread: on  Write thread: off
```
```
node0> sh ip bgp summary
BGP router identifier 10.0.0.10, local AS number 64512
RIB entries 11, using 1232 bytes of memory
Peers 3, using 13 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.11    4 64512      23      27        0    0    0 00:20:06        2
192.168.2.12    4 64513      25      27        0    0    0 00:20:02        1
192.168.3.13    4 64514      23      27        0    0    0 00:20:06        2

Total number of neighbors 3
```
```
node0> sh ip bgp prefix-list FILTER01-out
BGP table version is 0, local router ID is 10.0.0.10
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.0.0.10/32     0.0.0.0                  0         32768 i
* i192.168.1.0      192.168.1.11             0    100      0 i
*>                  0.0.0.0                  0         32768 i
*  192.168.2.0      192.168.2.12             0             0 64513 i
*>                  0.0.0.0                  0         32768 i

Total number of prefixes 3
```
License
-------

BSD

Author Information
------------------

Larry Smith Jr.
- @mrlesmithjr
- http://everythingshouldbevirtual.com
- mrlesmithjr [at] gmail.com

[Ansible]: <https://ansible.com>
[Vagrant]: <https://www.vagrantup.com/>
[VirtualBox]: <https://www.virtualbox.org/>
