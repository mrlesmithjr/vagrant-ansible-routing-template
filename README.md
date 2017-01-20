Purpose
=======

Spin up vagrant multi node routing environment and manage all nodes using Ansible.

Follow the "Hey I can DevOPS my Network too!" series on my blog at the following
link. http://everythingshouldbevirtual.com/hey-i-can-devops-my-network-too-intro
* Note: The above blog post is out of date until it is updated based on the new
configuration here.

This lab will spin up 7 nodes in a spine leaf simulation with BGP routing between
all nodes including the compute nodes. The compute nodes are running [Docker] so
you can experiment with running some containers in this environment as well. All
[Docker] compute nodes participate in a [Docker] swarm cluster as well.

* node0 - Spine
* node1|node2 - Leafs
* node3|node4 - Compute/Docker Swarm Managers
* node5|node6 - Compute/Docker Swarm Workers

Topology
--------
![Screenshot](spine-leaf-compute.png)

Each node has a management interface which is used to provision the nodes via
[Ansible], as well as on the compute nodes which are participating in the
[Docker] swarm cluster you can connect to the services running using the management
interface IP addresses.
* node0 - 192.168.250.10
* node1 - 192.168.250.11
* node2 - 192.168.250.12
* node3 - 192.168.250.13
* node4 - 192.168.250.14
* node5 - 192.168.250.15
* node6 - 192.168.250.16

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
quagga_interfaces:
  - int: 'enp0s9'
    configure: true
    method: 'static'
    address: '192.168.1.10'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
  - int: 'enp0s10'
    configure: true
    method: 'static'
    address: '192.168.2.10'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
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
quagga_interfaces:
  - int: 'enp0s9'
    configure: true
    method: 'static'
    address: '192.168.1.11'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
  - int: 'enp0s10'
    configure: true
    method: 'static'
    address: '192.168.10.11'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
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
quagga_interfaces:
  - int: 'enp0s9'
    configure: true
    method: 'static'
    address: '192.168.2.12'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
  - int: 'enp0s10'
    configure: true
    method: 'static'
    address: '192.168.20.12'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
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
quagga_interfaces:
  - int: 'enp0s9'
    configure: true
    method: 'static'
    address: '192.168.10.13'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
  - int: 'enp0s10'
    configure: true
    method: 'static'
    address: '192.168.30.13'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0
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
`host_vars/node4/quagga.yml`
```
---
quagga_interfaces:
  - int: 'enp0s9'
    configure: true
    method: 'static'
    address: '192.168.20.14'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0'
  - int: 'enp0s10'
    configure: true
    method: 'static'
    address: '192.168.40.14'
    # gateway: '10.1.1.1'
    cidr: '24'
    netmask: '255.255.255.0'
  #   addl_settings:
  #     - 'bond_master bond0
quagga_interfaces_lo:
  - int: "lo{{ ':' }}0"
    address: '10.4.4.14/32'
    method: 'static'
    configure: true
quagga_ospf_area_config:
  - network: '192.168.3.0/24'
    area: '{{ quagga_ospf_area }}'
quagga_ospf_routerid: '10.4.4.14'
```
Usage
-----

```
git clone https://github.com/mrlesmithjr/vagrant-ansible-routing-template.git
cd vagrant-ansible-routing-template
git checkout spine-leaf-compute
```

Spin up your environment
```
vagrant up
```

BGP
---

```
vagrant ssh node0
```
```
vagrant@node0:~$ ip route
default via 10.0.2.2 dev enp0s3
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15
192.168.1.0/24 dev enp0s9  proto kernel  scope link  src 192.168.1.10
192.168.2.0/24 dev enp0s10  proto kernel  scope link  src 192.168.2.10
192.168.10.0/24 via 192.168.1.11 dev enp0s9  proto zebra
192.168.20.0/24 via 192.168.2.12 dev enp0s10  proto zebra
192.168.30.0/24 via 192.168.1.11 dev enp0s9  proto zebra
192.168.40.0/24 via 192.168.2.12 dev enp0s10  proto zebra
192.168.50.0/24 via 192.168.1.11 dev enp0s9  proto zebra
192.168.60.0/24 via 192.168.2.12 dev enp0s10  proto zebra
192.168.250.0/24 dev enp0s8  proto kernel  scope link  src 192.168.250.10
```
Now enter the `vtysh` shell and run some additional validations:
```
sudo vtysh
```
```
node0# sh ip bgp
BGP table version is 0, local router ID is 10.0.0.10
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 192.168.1.0      0.0.0.0                  0         32768 i
*> 192.168.2.0      0.0.0.0                  0         32768 i
*> 192.168.10.0     192.168.1.11             0             0 64513 i
*> 192.168.20.0     192.168.2.12             0             0 64514 i
*> 192.168.30.0     192.168.1.11                           0 64513 i
*> 192.168.40.0     192.168.2.12                           0 64514 i
*> 192.168.50.0     192.168.1.11                           0 64513 i
*> 192.168.60.0     192.168.2.12                           0 64514 i

Total number of prefixes 8
```
```
node0# sh ip bgp neighbors
BGP neighbor is 192.168.1.11, remote AS 64513, local AS 64512, external link
 Description: "leaf-1-node1"
  BGP version 4, remote router ID 10.1.1.11
  BGP state = Established, up for 00:36:30
  Last read 00:00:29, hold time is 180, keepalive interval is 60 seconds
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
    Updates:                4          3
    Keepalives:            38         37
    Route Refresh:          0          0
    Capability:             0          0
    Total:                 43         41
  Minimum time between advertisement runs is 30 seconds

 For address family: IPv4 Unicast
  Inbound soft reconfiguration allowed
  NEXT_HOP is always this router
  Community attribute sent to this neighbor(both)
  2 accepted prefixes

  Connections established 1; dropped 0
  Last reset never
Local host: 192.168.1.10, Local port: 29690
Foreign host: 192.168.1.11, Foreign port: 179
Nexthop: 192.168.1.10
Nexthop global: fe80::a00:27ff:fe4f:f988
Nexthop local: ::
BGP connection: non shared network
Read thread: on  Write thread: off

BGP neighbor is 192.168.2.12, remote AS 64514, local AS 64512, external link
 Description: "leaf-2-node2"
  BGP version 4, remote router ID 10.2.2.12
  BGP state = Established, up for 00:36:30
  Last read 00:00:29, hold time is 180, keepalive interval is 60 seconds
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
    Updates:                4          3
    Keepalives:            38         37
    Route Refresh:          0          0
    Capability:             0          0
    Total:                 44         40
  Minimum time between advertisement runs is 30 seconds

 For address family: IPv4 Unicast
  Inbound soft reconfiguration allowed
  NEXT_HOP is always this router
  Community attribute sent to this neighbor(both)
  2 accepted prefixes

  Connections established 1; dropped 0
  Last reset never
Local host: 192.168.2.10, Local port: 179
Foreign host: 192.168.2.12, Foreign port: 9788
Nexthop: 192.168.2.10
Nexthop global: fe80::a00:27ff:fe02:6cb7
Nexthop local: ::
BGP connection: non shared network
Read thread: on  Write thread: off
```
```
node0# sh ip bgp summary
BGP router identifier 10.0.0.10, local AS number 64512
RIB entries 11, using 1232 bytes of memory
Peers 2, using 9136 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.11    4 64513      42      44        0    0    0 00:37:16        2
192.168.2.12    4 64514      41      45        0    0    0 00:37:16        2

Total number of neighbors 2
```
```
vagrant ssh node1
```
```
node1# sh ip bgp summary
BGP router identifier 10.1.1.11, local AS number 64513
RIB entries 15, using 1680 bytes of memory
Peers 3, using 13 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.10    4 64512      41      43        0    0    0 00:36:33        5
192.168.10.13   4 64513      39      45        0    0    0 00:36:40        1
192.168.10.15   4 64513      40      44        0    0    0 00:36:41        1

Total number of neighbors 3
```
```
node1# sh ip bgp
BGP table version is 0, local router ID is 10.1.1.11
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 192.168.1.0      192.168.1.10             0             0 64512 i
*> 192.168.2.0      192.168.1.10             0             0 64512 i
*> 192.168.10.0     0.0.0.0                  0         32768 i
*> 192.168.20.0     192.168.1.10                           0 64512 64514 i
*>i192.168.30.0     192.168.10.13            0    100      0 i
*> 192.168.40.0     192.168.1.10                           0 64512 64514 i
*>i192.168.50.0     192.168.10.15            0    100      0 i
*> 192.168.60.0     192.168.1.10                           0 64512 64514 i

Total number of prefixes 8
```

[Docker] Swarm
------------
This lab also has a fully functional [Docker] swarm cluster. The following nodes
participate in the cluster:
* node3 - [Docker] swarm manager
* node4 - [Docker] swarm manager
* node5 - [Docker] swarm worker
* node6 - [Docker] swarm worker

```
vagrant ssh node3
```
vagrant@node3:~$ sudo docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
63qtebcwjey5rr4u9xgckhi36 *  node3     Ready   Active        Leader
fvbwd68tz7vuc226958vuy96x    node6     Ready   Active
sfmhmsqaf2gsy4gidprbthbya    node4     Ready   Active        Reachable
zhvo00r0oyra9h435ydel4prj    node5     Ready   Active
```
```
vagrant@node3:~$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
7a885d70df5a        bridge              bridge              local
b60adc742a0b        docker_gwbridge     bridge              local
5b575c7af449        host                host                local
v0ysnbsxx14g        ingress             overlay             swarm
f8ea47296ee5        none                null                local
```

Create a web service running NGINX:
```
vagrant@node3:~$ sudo docker service create --name web -p 8080:80 nginx
y50ccsl5x8yct69q574g1ldrp
```
Validate the web service is running:
```
vagrant@node3:~$ sudo docker service ls
ID            NAME  MODE        REPLICAS  IMAGE
y50ccsl5x8yc  web   replicated  1/1       nginx:latest
```
You should now be able to connect to the web service on any one of the following
urls:
http://192.168.250.13:8080/
http://192.168.250.14:8080/
http://192.168.250.15:8080/
http://192.168.250.16:8080/

Now scale the web service to spin up additional containers:
```
vagrant@node3:~$ sudo docker service scale web=3
web scaled to 3
```
Validate that the service scaled up:
```
vagrant@node3:~$ sudo docker service ls
ID            NAME  MODE        REPLICAS  IMAGE
y50ccsl5x8yc  web   replicated  3/3       nginx:latest
```
Show the web service instances:
```
vagrant@node3:~$ sudo docker service ps web
ID            NAME   IMAGE         NODE   DESIRED STATE  CURRENT STATE               ERROR  PORTS
umgusnq162t4  web.1  nginx:latest  node6  Running        Running 11 minutes ago
tqefm5er127k  web.2  nginx:latest  node3  Running        Running about a minute ago
npnob1w5azf7  web.3  nginx:latest  node4  Running        Running 7 minutes ago
```
And if you were to connect to one of the containers and validate that you can
ping the spine `node0 - 192.168.1.10`:

Obtain one of the container ID's of the running web service
(replace 25ba8ad0a2b1 with your actual container id):
```
vagrant@node3:~$ sudo docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED             STATUS              PORTS               NAMES
25ba8ad0a2b1        nginx@sha256:33ff28a2763feccc1e1071a97960b7fef714d6e17e2d0ff573b74825d0049303   "nginx -g 'daemon ..."   4 minutes ago       Up 4 minutes        80/tcp, 443/tcp     web.2.tqefm5er127kv334y0f0eck2l
```
```
vagrant@node3:~$ sudo docker exec -it 25ba8ad0a2b1 bash
root@25ba8ad0a2b1:/# ping 192.168.1.10 -c 4
PING 192.168.1.10 (192.168.1.10): 56 data bytes
64 bytes from 192.168.1.10: icmp_seq=0 ttl=62 time=81.996 ms
64 bytes from 192.168.1.10: icmp_seq=1 ttl=62 time=80.626 ms
64 bytes from 192.168.1.10: icmp_seq=2 ttl=62 time=78.060 ms
64 bytes from 192.168.1.10: icmp_seq=3 ttl=62 time=76.859 ms
--- 192.168.1.10 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 76.859/79.385/81.996/2.031 ms
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
[Docker]: <https://www.docker.com/>
[Vagrant]: <https://www.vagrantup.com/>
[VirtualBox]: <https://www.virtualbox.org/>
