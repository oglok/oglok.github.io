---
layout: post
title: Install Cumulus Linux via ONIE
subtitle: on EdgeCore switch, but it doesn't matter
bigimg: /img/cumulus-linux.png
---

This quickstart tutorial will guide you through the steps to install Cumulus Linux via ONIE. ONIE (Open Network Install Environment) is a small operating system, pre-installed on bare metal network switches, that provides an environment for automated provisioning.

# Connect to the OOB console

In order to connect to the console, we would need to follow this link:

[https://www.conserver.com/docs/console.man.html](https://www.conserver.com/docs/console.man.html)

After installing conserver, you will need to specify some configuration on your .consolerc file:

~~~
config *{                                                                                                                                         
    master console.server.hostname;
    username your-username;
} 

~~~

We will connect to the console such:

~~~
console myToR.oot.lab.eng.rdu2.redhat.com

~~~

![Console](/img/cumus_onie/1.png "Console")


# Mount a TFTP server

We need to locate the Cumulus Linux bin file in a TFTP server. You need to get an account to download the image file and take into account you will have to choose architecture of your switch (x86, ARM, and also the ASICs). Please, if possible, pick a server within the same network as the switch is located. Otherwise, fetching the image will take longer and eventually time out.

Go to the server where you want to start the TFTP server and:

~~~
sudo yum install -y tftp-server
sudo systemctl start tftp
sudo systemctl status tftp
~~~

![TFTP](/img/cumus_onie/2.png "TFTP Server")


Copy the bin file to /var/lib/tftpboot (default folder for the tftp server):

sudo cp cumulus-linux-3.5.3-bcm-amd64.bin /var/lib/tftpboot/


# Install Cumulus Linux

Now that the Cumulus Linux is located in the TFTP server, ONIE will allow us to fetch the image and install the firmware.

First remember to disable discovery mode:

~~~
ONIE: Starting ONIE Service Discovery              
ONIE:/ # onie-discovery-stop 
discover: installer mode detected.                 
Stopping: discover... done.
~~~

And execute the installation command:

~~~
ONIE:/ # onie-nos-install tftp://x.x.x.x/cumulus-linux-3.5.3-bcm-amd64.bin
~~~


![Fetching](/img/cumus_onie/3.png "Fetching Image")

![Boot](/img/cumus_onie/4.png "Boot loader")

![Install](/img/cumus_onie/5.png "Installation")

![Login](/img/cumus_onie/6.png "Login")

The default username and password are:

~~~
cumulus/CumulusLinux!
~~~

Finally, a user guide for this version of Cumulus Linux can be found here:

[https://docs.cumulusnetworks.com/display/DOCS?preview=/7112747/7114050/Cumulus%20Linux%203.5.3%20User%20Guide.pdf](https://docs.cumulusnetworks.com/display/DOCS?preview=/7112747/7114050/Cumulus%20Linux%203.5.3%20User%20Guide.pdf)

