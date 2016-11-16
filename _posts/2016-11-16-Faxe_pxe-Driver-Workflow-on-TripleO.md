---
layout: post
title: How to use TripleO with fake_pxe driver
subtitle: ...and when to reboot servers
bigimg: /img/tripleo.png
---

TripleO is one of the various installers for Openstack out there in the community. TripleO receives its name from Openstack Over Openstack, and eventhough the architecture is a little bit complex, the main advantage is that it uses general services from Openstack (identity, images, networking, etc) to provision a production OpenStack.

Take a look at the following link to know more about TripleO:

[http://docs.openstack.org/developer/tripleo-docs/](http://docs.openstack.org/developer/tripleo-docs/)

Summarizing the workflow, first you need to install what we call the "undercloud" which is the provisioning machine (or VM). The undercloud will use Ironic in order to provision baremetal/virtual nodes. Usually, Ironic will access the IPMI interfaces for power management (PXE booting, power up/down, etc). It counts with a set of drivers for most of the important IPMI protocols such iDRAC, ILOM, etc. However, sometimes our HW is a little bit more "special" and Ironic doesn't support the IPMI driver for that kind of server. That's where the fake_pxe driver comes to play!

First of all, we need to create a json file with a kind of inventory of our target nodes. Something like this:

~~~
{
    "nodes":[
        {
            "mac":[
                "AA:AA:AA:AA:AA:AA"
            ],
            "arch":"x86_64",
            "pm_type":"fake_pxe"
        },
        {
            "mac":[
                "BB:BB:BB:BB:BB:BB"
            ],
            "arch":"x86_64",
            "pm_type":"fake_pxe"
        },
        {
            "mac":[
                "CC:CC:CC:CC:CC:CC"
            ],
            "arch":"x86_64",
            "pm_type":"fake_pxe"
        }
    ]
}

~~~

As you can see, we have specify some information for three nodes where OpenStack (the overcloud) will be deployed. That information basically is the MAC address of the provisioning inteface that will receive the DHCP messages, the architecture of the server and the type of driver: our fake_pxe!

Some important ideas to take into consideration:

* This driver does not do any power management, so no authentication is needed.
* It must be configured in the /etc/ironic/ironic.conf of the undercloud (see later)
* Target nodes must be rebooted manually in a specific workflow that will explain later.

So let's say we import the inventory json file to our Ironic database:

~~~
openstack baremetal import --json ~/instackenv.json
~~~

Assign the ramdisk and kernel images to those nodes:

~~~
openstack baremetal configure boot
~~~

And check the status:

~~~
openstack baremetal node list
~~~

There are two phases where Ironic plays an important role: introspection and deployment. This blogpost doesn't try to explain TripleO, but just where the workflow to make a successful deployment with fake_pxe.

For introspection, PXE booting must be configured and nodes need to be rebooted in order to start the process. That's pretty straigh forward.

The tricky process comes with the deployment. Before launching the deployment, nodes must be powered off. Let's show the state in Ironic database:

~~~
[stack@undercloud ~]$ openstack baremetal node list
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
| ce9937be-d2f9-4d1b-b3ae-016c128ed6af | None |                                      | power off   | available          | False       |
| 1b224050-45c1-46ee-9cf7-5df88479dfc9 | None |                                      | power off   | available          | False       |
| fa38b21a-8521-4ebf-8af8-abe7351a2bb1 | None |                                      | power off   | available          | False       |
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
~~~

You must configure once again in the power management system from your hardware to make a one time boot via PXE. Then power off your target nodes and launch deployment such:

~~~
openstack overcloud deploy ....
~~~

The undercloud will start creating a new Heat stack with lots of resources and very complex configurations. You need to monitor in another window the status of the baremetal nodes using the very same command (openstack baremetal node list). 

At some point, nodes will be in a "waiting call-back" state such:

~~~
[stack@undercloud ~]$ openstack baremetal node list
+--------------------------------------+------+--------------------------------------+-------------+----------------------+-------------+
| UUID                                 | Name | Instance UUID                        | Power State | Provisioning State   | Maintenance |
+--------------------------------------+------+--------------------------------------+-------------+----------------------+-------------+
| ce9937be-d2f9-4d1b-b3ae-016c128ed6af | None | e2119c85-fa1e-411d-8de3-7c307286639d | power on    | deploy wait-callback | False       |
| 1b224050-45c1-46ee-9cf7-5df88479dfc9 | None | 5ba04bac-8a71-484b-9f15-d8d56f0df876 | power on    | deploy wait-callback | False       |
| fa38b21a-8521-4ebf-8af8-abe7351a2bb1 | None | 02840b27-08e1-4e7e-b028-97b312dd1bd3 | power on    | deploy wait-callback | False       |
+--------------------------------------+------+--------------------------------------+-------------+----------------------+-------------+
~~~

POWER UP YOUR NODES! This is the first milestone where you will need to interact with your servers. Once the three nodes are up & running, the provisioning state will change to "Deploying" state. It will take a little bit of time but it will end up in "Active" state.

~~~
[stack@undercloud ~]$ openstack baremetal node list
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
| ce9937be-d2f9-4d1b-b3ae-016c128ed6af | None | e2119c85-fa1e-411d-8de3-7c307286639d | power on    | active             | False       |
| 1b224050-45c1-46ee-9cf7-5df88479dfc9 | None | 5ba04bac-8a71-484b-9f15-d8d56f0df876 | power on    | active             | False       |
| fa38b21a-8521-4ebf-8af8-abe7351a2bb1 | None | 02840b27-08e1-4e7e-b028-97b312dd1bd3 | power on    | active             | False       |
+--------------------------------------+------+--------------------------------------+-------------+--------------------+-------------+
~~~

The deployment process has powered off your nodes and it's waiting to perform configuration over the images that are installed in the servers. If you check the power status of your servers it will indicate: OFF. HURRY UP! POWER THEM UP AGAIN!

And that's all folks! Once the servers are back, the undercloud will call several puppet manifests to configure the different services in the overcloud (Nova, Neutron, Keystone, etc). If you have defined your network templates and environment files properly, you will get the precious "OVERCLOUD DEPLOYED" that we all love!
