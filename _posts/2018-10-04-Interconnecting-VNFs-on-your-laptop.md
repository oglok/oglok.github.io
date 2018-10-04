---
layout: post
title: Interconnecting VNFs on your laptop
subtitle: I love you, UDP unicast tunnels
bigimg: /img/tunnel.jpg
tags:
  nfvpe
category: nfvpe
---

For the past few weeks, I have been working on creating a virtualized environment that would allow me to test different configurations and topologies with Virtualized Network Functions. In this case, virtual routers. One of the challenges when working with this type of workloads in virtualized environments is to emulate the connections of the different ports in a switch/router to the rest of the infrastructure.

Libvirt provides different types of networking that are well-known:

- NAT forwarding
- Bridged networking
- PCI passthrough

The first two types of libvirt networks involved the usage of linux bridge and Iptables in order to get access to the external world from the VMs.
You can avoid these linux bridges attaching one of the physical interfaces of the host, directly to the virtual machine, however, we would need a lot of interfaces (physical NICs, macvlans, etc) to simulate the different ports in a virtualized switch. Furthermore, the traffic would be probably redirected outside our hosts, so this is a solution not very convenient.

# UDP Tunnels: that's right!

Libvirt provides many other types of interfaces. You can get further information in the following link:

[https://libvirt.org/formatdomain.html#elementsNICSUDP](https://libvirt.org/formatdomain.html#elementsNICSUDP)


The interesting part of this link is the following text:

~~~
A UDP unicast architecture provides a virtual network which enables connections between QEMU instances using QEMU's UDP infrastructure. The xml "source" address is the endpoint address to which the UDP socket packets will be sent from the host running QEMU. The xml "local" address is the address of the interface from which the UDP socket packets will originate from the QEMU host.
~~~

UDP unicast tunnels give us the possibility to emulate point to point connections among different virtual machines running on the same host,
which turns out to be a very smart solution to interconnect virtualized switches/routers with servers in any desired topology.

# Hands-on!!

Let's take a look at a great Ansible playbook from Peter Sprygada that I've modified a little bit.

[https://github.com/oglok/virtual_lab](https://github.com/oglok/virtual_lab)

You will need some requirements to run this playbook on your host, but with the following packages should be fine:

- python-pip
- ansible (via pip)
- libvirt-python
- git

In this case, the target machine will be just localhost, so we will run everything locally:

[https://github.com/oglok/virtual_lab/blob/master/inventory/hosts.dev](https://github.com/oglok/virtual_lab/blob/master/inventory/hosts.dev)

In order to run the playbook, it's just as easy as:

~~~
ansible-playbook -i inventory/hosts.dev playbooks/build.yaml
~~~

The goal of this playbook is to create the following topology:

![Topology](/img/basic_infra.png "Basic Topology")

which consists on four different virtual machines connected to two linux bridges that will give us SSH connectivity from the host.
You can modify the playbook to deploy whatever VNF you want, but as of now, this playbook would deploy two Cisco IOS and two Arista vEOS routers. For obvious reasons, the images for these VNFs are not uploaded since licenses are needed. But that's not important here, we could be using simple Linux boxes like Fedora or Centos with OpenVswitch on it, and emulate the same behaviour.

Let's take a look at the definition of one of the boxes located in the folder nodes.

[https://github.com/oglok/virtual_lab/blob/master/nodes/ios01.yaml](https://github.com/oglok/virtual_lab/blob/master/nodes/ios01.yaml)

~~~
---
ansible_network_os: ios
enabled: yes

vm_disk: csr1000v-universalk9.16.08.01a-serial.qcow2
vm_config: csr_config01.iso
hostname: ios01
console_port: 2005

connections:
  - type: udp
    s_port: 10501
    d_port: 10202

  - type: udp
    s_port: 10502
    d_port: 10602

  - type: udp
    s_port: 10503
    d_port: 10000

  - type: udp
    s_port: 10504
    d_port: 10000

netlab_mgmt_source_bridge: br0
mac_address: 52:54:00:cc:cc:01
~~~

You can guess some of the parameters used to deploy the VM, but check out the connections section.
We are defining a set of interfaces of type: udp, with some source port and destination port. You can choose whatever port you want here. Now you would be asking, hey! so how do I do the point to point connection?

No big deal, it's just a matter of swapping the s_port and d_port in a second box. So if we define ios2 with the following connections:

~~~
connections:
  - type: udp
    s_port: 10601
    d_port: 10102

  - type: udp
    s_port: 10602
    d_port: 10502

  - type: udp
    s_port: 10603
    d_port: 10000

  - type: udp
    s_port: 10604
    d_port: 10000
~~~

It will mean that the second interface of the first Cisco IOS will be directly connected to the second interface of Cisco IOS via a UDP tunnel. Subsequently, all the traffic sent in to one of the interfaces, will get out through that interface on the second box.

You can define as many ports as you want in those node definition files. Let's take a look at how the XML definition of the virtual machine looks like:

[https://github.com/oglok/virtual_lab/blob/master/roles/netlab/templates/ios/vm.xml.j2](https://github.com/oglok/virtual_lab/blob/master/roles/netlab/templates/ios/vm.xml.j2)

The XML shows three types of interfaces for the same VM:

~~~
    <interface type='bridge'>
      <mac address='{{ node.mac_address }}'/>
      <source bridge='{{ node.netlab_mgmt_source_bridge }}'/>
      <model type='virtio'/>
    </interface>
~~~
~~~
    \{% for link in node.connections | default([]) %\}
    \{% if link.type | default('udp') == 'udp' %\}
    <interface type='udp'>
      <source address='127.0.0.1' port='\{\{ link.s_port \}\}'>
        <local address='127.0.0.1' port='\{\{ link.d_port \}\}'/>
      </source>
      <model type='virtio'/>
    </interface>
    \{% endif %\}
    \{% endfor %\}
~~~
~~~
    <serial type='tcp'>
      <source mode='bind' host='0.0.0.0' service='{{ node.console_port }}' tls='no'/>
      <protocol type='telnet'/>
    </serial>
~~~


- The first interface will be bridged to one of the Linux bridges shown in the image above.
- The next section creates as many UDP tunnel endpoint interfaces as defined in the node definition file explained before, so in this case, there will be four of them.
- The last interface, is a serial port that will allow the administrator to connect to the console of the router via telnet. This keeps the type of operations that network engineering teams do on a daily basis.

It has been a lot of fun playing with this, and it gives a lot of flexibility to create different and complex topologies. And of course, what better tool to automate its deployment than Ansible?

If you have any question about the playbook, or any other subject, contact me via twitter, email, etc

[https://oglok.github.io/aboutme/](https://oglok.github.io/aboutme/)

