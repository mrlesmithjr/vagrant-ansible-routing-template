Purpose
=======

Spin up vagrant multi node routing environment and manage all nodes using Ansible.

Follow the "Hey I can DevOPS my Network too!" series on my blog at the following
link. http://everythingshouldbevirtual.com/hey-i-can-devops-my-network-too-intro
* Note: The above blog post is out of date until it is updated based on the new
configuration here.

Requirements
============

* [Ansible]
* [VirtualBox]
* [Vagrant]



Variable Definitions
====================

Usage
=====

```
git clone https://github.com/mrlesmithjr/vagrant-ansible-routing-template.git
cd vagrant-ansible-routing-template
```

Spin up your environment
```
vagrant up
```

If you define `quagga_enable_ospfd: true` and `quagga_config_ospfd: true` in
`group_vars/quagga-routers/quagga.yml` and then run the
`ansible-playbook -i hosts playbook.yml` your ip route should look something
similar to below:

```
vagrant@r1:/vagrant$ ip route
```
And if you were to telnet to the ospfd daemon and run sh ip ospf neighbors
```
r1# sh ip ospf neighbor
```
Additional sh ip ospf commands.
```
r1# sh ip ospf border-routers
```
```
r1# sh ip ospf database
```
```
r1# sh ip ospf interface
```
```
r1# sh ip ospf route
```

If you configure this environment and set `quagga_enable_bgpd: true` and then
run the playbook.yml your ip route should look something similar to below.
```
vagrant@r1:/vagrant$ ip route
```
If you connect to the bgpd daemon and run some commands they should look
similar to below.
```
telnet localhost 2605
```
quagga/quagga (username/password)
```
r1# sh run
```
```
r1# sh ip bgp
```
```
r1# sh ip bgp neighbors
```
```
 r1# sh ip bgp sum
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
