---
layout: post
title: Developing a TripleO composable service
subtitle: ...it's not that hard!
bigimg: /img/tripleo.png
tags:
  nfvpe
category: nfvpe
---

For the past months I've been integrating some services into TripleO. TripleO is an installer that leverages OpenStack services to deploy a whole OpenStack environment. There are loads of blog posts talking about what TripleO is so let's move on!

What it is important to know is that TripleO uses Heat templates to orchestrate the deployment, and puppet-openstack modules for installation and configuration of multiple services.

[https://wiki.openstack.org/wiki/Puppet](https://wiki.openstack.org/wiki/Puppet)

The example I am going to show here is to be able to deploy Neutron BGPVPN Driver with TripleO. BGP based VPNs are commonly offered by Carrier Service Providers to be used by enterprises.

[https://docs.openstack.org/developer/networking-bgpvpn/](https://docs.openstack.org/developer/networking-bgpvpn/)

There are four main projects to contribute to:

* puppet-neutron
* puppet-tripleo
* tripleo-heat-templates
* tripleo-puppet-elements

## Puppet Neutron

I decided to develop the BGPVPN into puppet-neutron for two reasons: one, it makes a lot of sense; two, it could be leveraged by all the installers that run puppet-neutron underneath.

Let me explain this. My main goal was to integrate networking-bgpvpn (that's the name of the project) into TripleO, right? So I could have written all the puppet code into puppet-tripleo (only used by TripleO). That's fine, but thinking big, I decided to included into puppet-neutron so for example Packstack could make use of it, with just a bit of effort. Reusability is good.

First of all, I wanted to summarize the steps I would need to run manually to install and configure networking-bgpvpn, and try to map those actions into puppet later on. These steps are:

* Install networking-bgpvpn project (from a RPM if possible)
* Configure service_plugins from /etc/neutron/neutron.conf
* Configure service_provider from /etc/neutron/networking_bgpvpn.conf
* Restart Neutron services


Let's have a look at the patch and explain some things:

[https://review.openstack.org/#/c/422051/](https://review.openstack.org/#/c/422051/)

Of course, you need to learn basic puppet to do this integration, but don't try to get it all at first... learning by doing and iterating over your mistakes can be quite efficient too.

I locate the main manifest under:

~~~
manifest/services/bgpvpn.pp
~~~

Reading this piece of code is quite easy to understand, but basically it ensures that a package is installed, it exposes a parameter called service_providers and it executes a Neutron command to sync the database. Voilá! That's what you do when you enable a Neutron plugin, right??

I've just told my manifest to configure service_providers... but where? I need puppet to configure /etc/neutron/networking_bgpvpn.conf!! Have a look at the following file:

~~~
lib/puppet/provider/neutron_bgpvpn_service_config/openstackconfig.rb
~~~

There it is!! You will need to write a provider (and also a type) where you will inherit all the properties of "openstackconfig" provider. Point to the config file you want to modify and there you go!

But wait, what about modifying service_plugin in /etc/neutron/neutron.conf?? That's not only used by me, and it's a very important file... Hope someone has exposed already that parameter for me to use. ;-)

[https://github.com/openstack/puppet-neutron/blob/master/manifests/init.pp#L40](https://github.com/openstack/puppet-neutron/blob/master/manifests/init.pp#L40)

Most of the main Neutron parameters are already exposed, so spend some time checking what it is available, and what you need.

The rest of the files under the folder spec, are puppet unit tests written in rspec. You need to write unit tests for each new component you have created, like type, provider and class. To be honest, it is still kind of black magic to me. Puppet purists say that you have to write your rsec tests before the manifests, like a specification of what you want to do... but I didn't do that... It seemed to me kind of weird since you need to know some of the variables in advance.

## Puppet TripleO

See: [https://review.openstack.org/#/c/427669/](https://review.openstack.org/#/c/427669/)

From my point of view, Puppet TripleO is the core of the installer. It configures and orchestrates the deployment leveraging all the goods and resources that Puppet brings. However, if what we want it's just to integrate a service like the Neutron BGPVPN driver, Puppet TripleO needs a connector from Heat templates to Puppet Neutron manifests. 

~~~
class tripleo::profile::base::neutron::bgpvpn (
  $step = hiera('step'),
) {
  if $step >= 4 {
    include ::tripleo::profile::base::neutron
    include ::neutron::services::bgpvpn
  }
}
~~~

That's it! Importing TripleO Neutron class and the one created in the previous section! Puppet TripleO defines steps where different configurations will be applied. Usually, extra services like this will be configured after step 4. It's not really easy to guess what every step does and it's not documented anywhere. So it's just a matter of getting reviews from somebody and learn of your mistakes.

## TripleO Heat Templates

[https://review.openstack.org/#/c/428089/](https://review.openstack.org/#/c/428089/)

One of the mysteries for me digging into this project was:

How the hell Heat templates are connected to the puppet variables within a class?

I really saw the light at the end of the tunnel looking when I saw this chunk of code:

~~~
outputs:
  role_data:
    description: Role data for the BGPVPN role.
    value:
      service_name: neutron_bgpvpn_api
      config_settings:
        neutron::services::bgpvpn::service_providers: {get_param: BgpvpnServiceProvider}
      step_config: |
        include ::tripleo::profile::base::neutron::bgpvpn
~~~

This code located in puppet/services/neutron-bgpvpn-api.yaml showed me how in a YAML file, I could reference the parameter I exposed in the bgpvpn.pp manifest.
So under the config_settings section, you can pass a value to all these parameters that you have written in your puppet manifest. After seeing this I could sleep and not dream about TripleO's owl.

At the beginning of this file, you will see other three parameters: ServiceNetMap, DefaultPasswords and EndpointMap. Well, these are mandatory parameters, but I didn't have the time to figure out what they do/mean. 

The other important file to create in this project is the environment file. Check environments/neutron-bgpvpn.yaml.

~~~
resource_registry:
  OS::TripleO::Services::NeutronBgpvpnApi: ../puppet/services/neutron-bgpvpn-api.yaml

parameter_defaults:
  NeutronServicePlugins: 'networking_bgpvpn.neutron.services.plugin.BGPVPNPlugin'
  BgpvpnServiceProvider: 'BGPVPN:Dummy:networking_bgpvpn.neutron.services.service_drivers.driver_api.BGPVPNDriver:default'
~~~

This file contains all the parameters that the user must configure specifically to his scenario. In order to make Neutron BGPVPN driver work, we have to configure service_plugin parameter in /etc/neutron/neutron.conf and service_provider in /etc/neutron/networking_bgpvpn.conf. TripleO will do this for the user (with your wonderfull service integration).

The rest of the logic is pretty simple. Locate your service in the role you need (controller, compute, etc...) by filling roles_data.yaml and map define if it should be a default service or not in overcloud-resource-registry-puppet.j2.yaml.

~~~
OS::TripleO::Services::NeutronBgpvpnApi: OS::Heat::None
~~~

This means NeutronBgpVpnApi should not be enabled by default. That's why in environments/neutron-bgpvpn.yaml we point the service to the proper puppet service neutron-bgpvpn-api.yaml.


## TripleO Puppet Elements

TripleO uses the very same image in the overcloud for all the different roles. It is just puppet changing configurations to actually differentiates the nodes as controllers, computes, etc.

For that reason, we need to include the package that needs to be installed in the corresponding element. This needs no explanation...

[https://review.openstack.org/#/c/404119/](https://review.openstack.org/#/c/404119/)


## Conclusion

TripleO is taugh to learn. There are many components to be orchestrated and troubleshooting could be a nightmare. However, there is a great community behind which can help you figure out your way, answer some questions and give advice. At the end, a service integration into TripleO doesn't seem so mysterious and I hope this tutorial helps other developers facing such a task.

Gracias!
