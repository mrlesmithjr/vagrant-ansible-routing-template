Purpose
=======

Spin up vagrant multi node routing environment and manage all nodes using Ansible. Upon spinning up vagrant will provision each node using Ansible to bootstrap the nodes. During the bootstrap each node will have a respective host_vars configuration file which will be updated with their eth1 address and ssh key file location. This allows you to run Ansible plays from within your HostOS or within any of your vagrant nodes.

Requirements
============

The following packages must be installed on your Host you intend on running all of this from. If Ansible is not available for your OS (Windows) You can modify the following lines in the Vagrantfile.

From:
````
#  config.vm.provision :shell, path: "bootstrap.sh"
#      node.vm.provision :shell, path: "bootstrap_ansible.sh"

config.vm.provision :ansible do |ansible|
  ansible.playbook = "bootstrap.yml"
end
````
To:
````
  config.vm.provision :shell, path: "bootstrap.sh"
      node.vm.provision :shell, path: "bootstrap_ansible.sh"

#config.vm.provision :ansible do |ansible|
#  ansible.playbook = "bootstrap.yml"
#end
````

Ansible (http://www.ansible.com/home)

VirtualBox (https://www.virtualbox.org/)

Vagrant (https://www.vagrantup.com/)



Variable Definitions
====================
````
nodes.yml
````
Define the nodes to spin up
````
---
- name: r1
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.101  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.12.11
      desc: 01-to-02
    - ip: 192.168.14.11
      desc: 01-to-04
    - ip: 192.168.31.11
      desc: 03-to-01
    - ip: 1.1.1.10
      desc: 'Network to Advertise'
- name: r2
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.102  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.23.12
      desc: 02-to-03
    - ip: 192.168.12.12
      desc: 01-to-02
    - ip: 2.2.2.10
      desc: 'Network to Advertise'
- name: r3
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.103  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.31.13
      desc: 03-to-01
    - ip: 192.168.23.13
      desc: 02-to-03
    - ip: 3.3.3.10
      desc: 'Network to Advertise'
- name: r4
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.104  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.14.14
      desc: 01-to-04
    - ip: 192.168.31.14
      desc: 03-to-01
    - ip: 192.168.41.14
      desc: utopia
    - ip: 4.4.4.10
      desc: 'Network to Advertise'
````

Provisions nodes by bootstrapping using Ansible
````
bootstrap.yml
````
Bootstrap Playbook
````
---
- hosts: all
  remote_user: vagrant
  sudo: yes
  vars:
    - galaxy_roles:
      - mrlesmithjr.bootstrap
      - mrlesmithjr.base
      - mrlesmithjr.quagga
    - install_galaxy_roles: true
    - ssh_key_path: '.vagrant/machines/{{ inventory_hostname }}/virtualbox/private_key'
    - update_host_vars: true
  roles:
  tasks:
    - name: updating apt cache
      apt: update_cache=yes cache_valid_time=3600
      when: ansible_os_family == "Debian"

    - name: installing ansible pre-reqs
      apt: name={{ item }} state=present
      with_items:
        - python-pip
        - python-dev
      when: ansible_os_family == "Debian"

    - name: adding ansible ppa
      apt_repository: repo='ppa:ansible/ansible'
      when: ansible_os_family == "Debian"

    - name: installing ansible
      apt: name=ansible state=latest
      when: ansible_os_family == "Debian"

    - name: installing ansible-galaxy roles
      shell: ansible-galaxy install {{ item }} --force
      with_items: galaxy_roles
      when: install_galaxy_roles is defined and install_galaxy_roles

    - name: ensuring host file exists in host_vars
      stat: path=./host_vars/{{ inventory_hostname }}
      delegate_to: localhost
      register: host_var
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: creating missing host_vars
      file: path=./host_vars/{{ inventory_hostname }} state=touch
      delegate_to: localhost
      sudo: false
      when: not host_var.stat.exists

    - name: updating ansible_ssh_port
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_port{{ ':' }}" line="ansible_ssh_port{{ ':' }} 22"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: updating ansible_ssh_host
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_host{{ ':' }}" line="ansible_ssh_host{{ ':' }} {{ ansible_eth1.ipv4.address }}"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: updating ansible_ssh_key
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_private_key_file{{ ':' }}" line="ansible_ssh_private_key_file{{ ':' }} {{ ssh_key_path }}"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: ensuring host_vars is yaml formatted
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="---" line="---" insertbefore=BOF
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars
````
````
group_vars/quagga-routers
````
Quagga Routers group configurations in default state
````
---
config_quagga: true
quagga_bgp_router_configs:
  - name: r1
    local_as: 123
    router_id: 1.1.1.1
    neighbors:
      - neighbor: 192.168.12.12
        remote_as: 123
      - neighbor: 192.168.31.13
        remote_as: 123
      - neighbor: 192.168.14.14
        remote_as: 141
  - name: r2
    local_as: 123
    router_id: 2.2.2.2
    neighbors:
      - neighbor: 192.168.12.11
        remote_as: 123
      - neighbor: 192.168.23.13
        remote_as: 123
  - name: r3
    local_as: 123
    router_id: 3.3.3.3
    neighbors:
      - neighbor: 192.168.23.12
        remote_as: 123
      - neighbor: 192.168.31.11
        remote_as: 123
  - name: r4
    local_as: 141
    router_id: 4.4.4.4
    neighbors:
      - neighbor: 192.168.14.11
        remote_as: 123
quagga_enable_bgpd: false
quagga_enable_ospfd: false
quagga_enable_password: quagga
quagga_ospf_routerid: '{{ ansible_eth1.ipv4.address }}'
quagga_password: quagga
````
````
playbook.yml
````
Provisioning playbook
````
---
- hosts: quagga-routers
  remote_user: vagrant
  sudo: true
  vars:
  roles:
    - mrlesmithjr.quagga
  tasks:
    - name: installing packages
      apt: name={{ item }} state=present
      with_items:
        - traceroute
````


Usage
=====

````
git clone https://github.com/mrlesmithjr/vagrant-ansible-routing-template.git
cd vagrant-ansible-routing-template
````
Update/modify nodes.yml to reflect your desired nodes to spin up and also update/modify group_vars/quagga-routers to reflect changes.

Spin up your environment
````
vagrant up
````

To run ansible from within Vagrant nodes (Ex. site.yml)
````
vagrant ssh r1 # or r2,r3, r4; all work
cd /vagrant
````
modify nodes.yml to reflect your desired routers and configurations and then run the playbook.yml playbook.
````
ansible-playbook -i hosts playbook.yml
````

To install ansible-galaxy roles within your HostOS
````
ansible-galaxy install mrlesmithjr.bootstrap
ansible-galaxy install mrlesmithjr.base
````

License
-------

BSD

Author Information
------------------

Larry Smith Jr.
- @mrlesmithjr
- http://everythingshouldbevirtual.com
- mrlesmithjr [at] gmail.com
