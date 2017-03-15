---
layout: post
title: Test puppet modules in a Docker
subtitle: ...and don't understand Ruby
bigimg: /img/puppet.jpg
tags:
  nfvpe
category: nfvpe
---

Everytime I need to test something that requires installing packages o messing up my laptop's configuration, I tend to create a development environment.
By development environment I mean any tool that isolates somehow the process I want to run and it doesn't screw my OS. Depending on the requirements, I choose LXC, KVM, Vagrant, Docker...

My current project is the integration of BGPVPN Neutron Service Plugin into TripleO. In my previous post, I talked a little bit about TripleO so I won't back to it. Just let me say that it is
Red Hat's main installer for Openstack. Underneath, TripleO uses puppet and some Heat templates which means I've had to start digging into Puppet.

I'm not a devops guy, so I'm not into these configuration management tools, but I have used Ansible in the past, and I really found it very easy to learn (at least the basics).
Puppet vs Ansible? Hey, let's don't do that... not again please!

Puppet is written in Ruby, and in order to define a custom type or custom provider, you have to learn a little bit of Ruby, if that's even possible! :-P
Before submiting my patches into OpenStack puppet-neutron project, I've managed to test the manifest locally, but to be honest, I didn't know how to run unit tests.

One of my colleagues sent me the following link (Thanks Luke!):

[http://docs.openstack.org/developer/puppet-openstack-guide/testing.html](http://docs.openstack.org/developer/puppet-openstack-guide/testing.html)

So I decided to run a Docker container where I could learn how to execute these unit tests. Waiting for Openstack's CI pipeline to give the results of these tests was making me sick, so this effort was good for OpenStack infrastructure as well as for my own sake. :)

After some iterations I've created a Dockerfile with all the dependencies needed for running ruby unit tests. You can find it in my github:

[https://github.com/oglok/myDockerfiles/blob/master/Dockerfile_puppet_spec](https://github.com/oglok/myDockerfiles/blob/master/Dockerfile_puppet_spec)

Let's have a look at it:

~~~

FROM centos:latest
MAINTAINER Ricardo Noriega De Soto <rnoriega@redhat.com >
LABEL version="0.1" description="Test Container"

# Packaged dependencies
RUN yum update -y && \
yum install git -y && \
yum install bundle rubygem-bundler ruby-devel zlib-devel -y && \
yum groupinstall "Development Tools" -y 
~~~

Basically this Dockerfile specifies to use the latest version of CentOS and install git, ruby and some compiling tools.
Go to the folder where you have copied this file. If you don't want to specify the file, just name it as Dockerfile.
Let's build the image:

~~~
docker build -t puppet_test .
~~~

-t indicates the name of the image you are building. It will be used later to run your container. The building process will download centos docker image, and perform the installation of those packages. Once it's done, let's run our container. I usually tend to run it in interactive mode and get inside the box:

~~~
docker run -t -i puppet_test /bin/bash
~~~

I'm in! That's what you would say after running this command. Let's do an example now. The puppet code I've been writing for the past couple of weeks is located in the puppet-neutron project. Clone the repo of the project and cherry-pick your patch:

~~~
git clone https://github.com/openstack/puppet-neutron.git
cd puppet-neutron
git fetch https://git.openstack.org/openstack/puppet-neutron refs/changes/51/422051/14 && git cherry-pick FETCH_HEAD
~~~

So you have the master branch of puppet-neutron with your patch in it. In order to run your unit test, follow the guide from the link.

~~~
mkdir vendor
export GEM_HOME=vendor
bundle install
~~~

This will install all the bundles (components) from rubygem repository. Without the packages installed during the docker build, this last command wouldn't work.
Now, you can run different type of tests:

~~~
bundle exec rake lint # Run puppet-lint
bundle exec rake syntax # Syntax check Puppet manifests and templates
bundle exec rake spec # Run spec tests in a clean fixtures directory
bundle exec rake acceptance # Run acceptance tests
~~~

If you get errors, modify the code or the tests and re-run these commands.

That's all folks!
