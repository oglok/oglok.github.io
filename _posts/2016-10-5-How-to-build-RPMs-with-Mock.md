---
layout: post
title: How to Build RPMs (with Mock)
subtitle: ...and contribute to RDO
bigimg: /img/rpm.jpg
---

For the past weeks I've been going through the process of packaging with the intention of contributing to RDO Community.
RDO community is in charge of packaging OpenStack for RPM-based environments such RHEL (Red Hat Enterprise Linux), CentOS and Fedora.
Pushing a new package into RDO requires some steps that are well documented in the following link:

[https://www.rdoproject.org/documentation/rdo-packaging/#how-to-add-a-new-package-to-rdo-trunk](https://www.rdoproject.org/documentation/rdo-packaging/#how-to-add-a-new-package-to-rdo-trunk)

However, I'm going to explain just part of the process that I've followed and I will focus more on the building using a great tool such Mock.
In a future post, I'll be explaining how to use DLRN, which is the OpenStack project which automates the RPM creation for many different projects!

First of all, open a bugzilla in [http://bugzilla.redhat.com](http://bugzilla.redhat.com) with the following fields:

*Product: RDO
*Component: Package Review
*Summary: New package: $NAME
*Comments: You have to add two files there the spec and the src.rpm

##HOW TO CREATE THE SPEC

If there is already a python package that could be installed via pip, there is a nice software that makes the spec file for you. For instance:

~~~
pyp2rpm -p 2 networking-bgpvpn
~~~

The output of this tools is a nice start, but it will require some changes. RDO maintainers have a set of templates and examples that are very useful:

https://github.com/openstack-packages/openstack-example-spec

One best practice would be to separate tests into a different package and set their execution as a condition to let the package get build. As you can see in the following file (https://github.com/openstack-packages/openstack-example-spec/blob/master/openstack-example-tests.spec) the checking condition will look like this:

~~~
%check
%{__python2} setup.py test
~~~

Every project will use a different tool to run the unit tests. Check the code in order to see which tool is being used (testr, ostestr, etc…) Many Openstack projects use tox as testing tool, so check the tox.ini file in order to see which python instruction has to be located under %check primitive.

To make a new package for tests, check the following instructions:


~~~
%package -n python2-%{library}-tests
Summary:    OpenStack example library tests
Requires:   python2-%{library} = %{version}-%{release}

%description -n python2-%{library}-tests
OpenStack example library.

This package contains the example library test files.
~~~

This will create another RPM file just for tests. In that case, we want to avoid these files to be included in the main package, so an exclude directive has to be included:

~~~
%files -n python2-%{library}
%license LICENSE
%{python2_sitelib}/%{module}
%{python2_sitelib}/%{module}-*.egg-info
%exclude %{python2_sitelib}/%{module}/tests
~~~


Being networking-bgpvpn the name of the package in pip repositories, or just locate the path of the source code. Once you have that spec file, upload it somewhere and add it to the comments like: 

Spec file: $URL_SPEC

##HOW TO CREATE THE SRC.RPM

Next step would be to create the srpm file. In order to do that, we recommend to use the tool “mock”:

~~~
sudo yum install mock
~~~

Delorean is the packaging project within the OpenStack framework. Our main goal besides building an RPM, it’s to get that package into RDO community, so it has to comply with Delorean directives.
Add your user to the mock group:

~~~
usermod -a -G mock [User name]
~~~

Copy the following configuration file to /etc/mock:

https://gist.githubusercontent.com/lukehinds/856cb55b8f5b480d9e641f8812749178/raw/97f813e30c89090247e5eb4d5bdca2c087b377a1/dlrn.cfg 

“Mock” is able to build rpms to a specific target such Fedora, CentOS, etc. With this config file, you are creating a new target platform named dlrn (Delorean).

In order to create the SRPM file out of the spec, you will also need the source code. There are two main options to do this. If you want to build the package out of master branch of the project, clone the code first and make a tar.gz out of it using sdist:

~~~
git clone https://github.com/openstack/networking-bgpvpn.git
cd networking-bgpvpn/
python setup.py sdist
~~~

The tarball will be located in the dist folder within the project directory.

If you want to use a stable branch or release version, get the tarball file from the following link:

http://tarballs.openstack.org/


Another location for tarballs when not in openstack, is https://pypi.python.org/pypi 

Let’s create the srpm file. To do that execute the following command:

~~~
mock --buildsrpm --spec=python-networking-bgpvpn.spec \
     --sources=networking-bgpvpn-4.0.1.tar.gz -r dlrn
~~~

If only a src rpm is generated, try to run again using --rebuild. This command will allow us to identify building/installation issues in an early phase.

~~~
mock --rebuild -r dlrn \ python-networking-bgpvpn-4.0.1-01.el7.centos.src.rpm
~~~

	
The output of this execution is the srpm (/var/lib/mock/dlrn-centos7-x86_64/result/). Both the spec and the srpm file must be added as comments in the bugzilla 

Spec file: $URL_SPEC
SRPM file: $URL_SRPM

##CHECK ERRORS DOING A REBUILD

pypi2rpm tool is very useful to create a first draft of the spec file, but it introduces some errors that must be fixed. A rebuild indicating the target platform and the recently build srpm will do the job. I’d recommend to copy the srpm to the local folder to have a copy of it:

~~~
cp /var/lib/mock/dlrn-centos7-x86_64/result/python-networking-bgpvpn-4.0.1-1.el7.centos.src.rpm $LOCAL_FOLDER
~~~

~~~
mock --rebuild -r dlrn \
python-networking-bgpvpn-4.0.1-1.el7.centos.src.rpm
~~~

Now it’s time to fix some issues. Let’s put an example. The following BuildRequires have been written by the pypi2rpm tool:

~~~
BuildConflicts: python-oslosphinx = 3.4.0
BuildConflicts: python-sphinx = 1.2.0
BuildConflicts: python-sphinx = 1.3b1
BuildRequires:  python-coverage >= 3.6
BuildRequires:  python-discover
BuildRequires:  python-hacking < 0.11
BuildRequires:  python-hacking >= 0.10.0
BuildRequires:  python-oslosphinx >= 2.5.0
BuildRequires:  python-oslotest >= 1.10.0
BuildRequires:  python-pbr >= 1.8
BuildRequires:  python-setuptools
BuildRequires:  python-sphinx < 1.3
BuildRequires:  python-sphinx >= 1.1.2
BuildRequires:  python-subunit >= 0.0.18
BuildRequires:  python-testrepository >= 0.0.18
BuildRequires:  python-testscenarios >= 0.4
BuildRequires:  python-testtools >= 1.4.0
BuildRequires:  python2-devel
BuildRequires:  python-sphinx
~~~

Python-oslosphinx does not exist as rpm package within Centos repos, but it’s just a typo. The package exists as python-oslo-sphinx.

If you want to check for existing packages, here it’s a good link:

http://buildlogs.centos.org/centos/7/cloud/x86_64/openstack-newton/

If it’s not in CentOS, but you find the RPM in Fedora repo (https://admin.fedoraproject.org/pkgdb/) just include that dependency in a local repo, and indicate that in the bugzilla.

About python-hacking, it is recommended not to put requirements over top versions. Just remove them.

Another good practice is to include the following MACRO at the beginning of the spec file:

~~~ 
%global pypi_name networking-bgpvpn                                                                                                                                                                     
%{!?upstream_version: %global upstream_version %{version}%{?milestone}}  
~~~

This macro will take the upstream version to name you package. Pypi2rpm tool will put the pypi_version macro, so you will have to replace it manually:

~~~
Replace: 
Source0:        https://files.pythonhosted.org/packages/source/n/%{pypi_name}/%{pypi_name}-%{version}.tar.gz
By: 
Source0:        https://files.pythonhosted.org/packages/source/n/%{pypi_name}/%{pypi_name}-%{upstream_version}.tar.gz
~~~

Another example:

	Replace:
~~~
Requires:   python-%{pypi_name} = %{version}-%{release}
~~~
	By:
~~~
Requires:   python-%{pypi_name} = %{upstream_version}-%{release}
~~~

##MOCK CHROOT

During the rebuild process of the package, mock will create a chroot where the installation of the package will happen. This is an isolated environment where all dependencies of our project will be installed, and tests will be executed. Sometimes, the result of tests executed manually or during rebuild process can differ. This unexpected behaviour could be caused by different versions of our dependencies. In order to get into the chroot environment, mock provides an option called shell:

~~~
mock -r dlrn shell                                                                                                                                        

INFO: mock.py version 1.2.20 starting (python version = 3.5.1)...
Start: init plugins
INFO: selinux enabled
Finish: init plugins
Start: run
Start: chroot init
INFO: calling preinit hooks
INFO: enabled root cache
INFO: enabled yum cache
Start: cleaning yum metadata
Finish: cleaning yum metadata
Finish: chroot init
Start: shell
<mock-chroot> sh-4.2# 
~~~


There you can do anything you want, like install pip and check the list of installed packages:

~~~
<mock-chroot> sh-4.2# easy_install pip
Searching for pip
Reading https://pypi.python.org/simple/pip/
Best match: pip 8.1.2
~~~

~~~
<mock-chroot> sh-4.2# pip list
~~~


Check versions of the dependencies listed on the spec file of your local environment and compare them to the chroot environment. This could cause some troubles during package building.

In order to manually run the tests within the chroot environment, we will have to locate the source code. Usually, the code is located under the following path:

~~~
/builddir/build/BUILD/$PROJECT_NAME
~~~

There you can run whatever is needed such:

~~~
python setup.py testr
~~~

Check how many tests have failed and why:

~~~
testr failing
~~~

Or whatever tool is being used for testing.

##COMMENTS

It is obvious than a couple of weeks working on this does not make you an expert whatsoever and of course, many errors and issues will not be covered here, but this is just my experience.
I hope it will help and you can drop me an email if you have any question about the process. See you in the next post!
