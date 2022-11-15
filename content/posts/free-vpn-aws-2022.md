+++
title = "Run a Free VPN Server on AWS (2022 Edition)"
description = "Run OpenVPN on an AWS EC2 instance"
images = ["/posts/vpn-2022/vpn-splash.jpg"]
date = 2022-11-15T10:48:50-08:00
draft = false
categories = [
  "DevOps",
]
series = ["vpn"]
tags = [
  "aws",
  "vpn",
  "security",
  "firewall",
  "hacking",
]
+++

This tutorial will take you through the steps of setting up an EC2 instance running OpenVPN server. It will then cover how to grant and revoke access to your VPN server.

{{< figure src="/posts/vpn-2022/vpn-splash.jpg" alt="Free VPN on AWS" align="center" attr="Image credit: [Privecstasy](https://unsplash.com/@privecstasy) on [Unsplash](https://unsplash.com)" >}}

AWS has an awesome firewall built into its core services, which can easily be used to make sure that only certain ports are open to the outside world. One extra step that we can take is to run a VPN server that serves as the gateway to our protected AWS resource, e.g. EC2, RDS, etc... We can then shutdown direct access to our AWS resources just by revoking access via our VPN server. This is very useful if you need to revoke access for a former employee.

AWS now offers a [managed VPN Service](https://aws.amazon.com/vpn/), but this service costs at least $72 a month and is even more expensive when your VPN serves a lot of traffic and users. A lot of smaller organizations don't need all the features of the managed service and instead can run their own VPN server for just the cost of an EC2 instance. You can even use the free tier so that you don't have to pay anything for the EC2 instance for the 1st year.

### Step 1 — Create the VPN Security Group

Overview: security groups allow your servers to communicate with each other in a private cloud while exposing specific ports to the world. We are going to create a security group to allow VPN access to our VPN server. We will assume that all your other EC2 instances are members of the default security group and that the default security group does not allow access from the outside world.

Log in at [https://aws.amazon.com](https://aws.amazon.com), type _EC2_ in the search box, and click on the target to go to the EC2 Dashboard.

From the EC2 dashboard, click _Security Groups_:

{{< figure src="/posts/vpn-2022/security-groups.png" alt="Security Groups" align="center" attr="Image credit: Author" >}}

Click _Create security group_:

{{< figure src="/posts/vpn-2022/create-security-group.png" alt="Create Security Group" align="center" attr="Image credit: Author" >}}

Enter a name and description of _vpn_ and specify inbound rules on ports _22_, _443_, _943_, and _1194_. Note: the protocol for port _1194_ is **UDP**.

{{< figure src="/posts/vpn-2022/inbound-rules.png" alt="Ports" align="center" attr="Image credit: Author" >}}

Note: if the IP addresses that your team uses are static then you can add yet another layer of security by specifying an IP address range in the _Source_ of your rules. However, you’ll want to leave the _Source_ open to anywhere if you want your team to be able to connect from any IP as they may be working from a hotel, home, cafe, etc…

### Step 2 — Create the EC2 Instance

Return to the EC2 Dashboard and then click _Launch instances_:

{{< figure src="/posts/vpn-2022/launch-instances.png" alt="Launch instances" align="center" attr="Image credit: Author" >}}

Select Ubuntu (you can of course select almost any other OS that runs OpenVPN, but this tutorial is tailored to Ubuntu)

{{< figure src="/posts/vpn-2022/ubuntu-22.png" alt="Ubuntu 22" align="center" attr="Image credit: Author" >}}

Select `t2.nano`:

{{< figure src="/posts/vpn-2022/t2-nano.png" alt="t2 nano" align="center" attr="Image credit: Author" >}}

In the _Network settings_ section, select your default VPC and disable the auto-assign public IP option. Then, select both your default security group and the security group that you created above for the VPN.

{{< figure src="/posts/vpn-2022/network-settings.png" alt="Edit security groups" align="center" attr="Image credit: Author" >}}

Click _Launch instance_

### Step 3 — Disable Source/Destination Check

From the list of instances, select the VPN instance and then _Actions -> Networking -> Change source/destination check_ from the drop down menu.

{{< figure src="/posts/vpn-2022/instance-source-destination.png" alt="Instance source destination check" align="center" attr="Image credit: Author" >}}

Select _Stop_ and click _Save_. This is needed as otherwise, your VPN server will not be able to connect to your other AWS resources.

{{< figure src="/posts/vpn-2022/edit-source-destination.png" alt="Edit source destination check" align="center" attr="Image credit: Author" >}}

### Step 4 — Create an Elastic IP Address

Overview: when an EC2 instance is stopped and restarted, the Public IP address changes. We want the IP address of our VPN Server to remain static so we’ll use an Elastic IP Address.

From the E2C Dashboard, select _Elastic IPs_:

{{< figure src="/posts/vpn-2022/elastic-ips-menu.png" alt="Elastic IPs" align="center" attr="Image credit: Author" >}}

Click _Allocate Elastic IP address_:

{{< figure src="/posts/vpn-2022/allocate-elastic-ip.png" alt="Allocate Elastic IP address" align="center" attr="Image credit: Author" >}}

Make a note of your new Elastic IP address as this will be the Public IP Address of your VPN server. We will refer to this address later as _PUBLIC-IP-OF-VPN-SERVER_.

Select the IP address you just created and click _Associate Elastic IP address_:

{{< figure src="/posts/vpn-2022/associate-elastic-ip.png" alt="Associate Elastic IP address" align="center" attr="Image credit: Author" >}}

Then select the Elastic IP and click _Associate address_ from the drop down menu.

Select the EC2 instance you created above and click _Associate_:

{{< figure src="/posts/vpn-2022/ec2-elastic-ip.png" alt="EC2 Elastic IP" align="center" attr="Image credit: Author" >}}

### Step 5— Install and Configure the OpenVPN Server

SSH into your VPN server:

```
$ ssh ubuntu@PUBLIC-IP-OF-VPN-SERVER
```

Download our helper scripts and set up a default config:

```
$ git clone https://github.com/redgeoff/openvpn-server-vagrant
$ cd openvpn-server-vagrant
$ cp config-default.sh config.sh
```

Edit config.sh and enter in your configuration. Note: PUBLIC_IP should be equal to the Elastic IP Address that you created above.

```
$ nano config.sh
```

Switch to root

```
$ sudo su -
```

You'll now update Ubuntu. Note: you will be prompted several times and when you do, just press the _Enter_ key.

```
$ /home/ubuntu/openvpn-server-vagrant/ubuntu.sh
```

Now, install OpenVPN. Note: you will be prompted several times and when you do, just press the _Enter_ key.

```
$ /home/ubuntu/openvpn-server-vagrant/openvpn.sh
```

At this point, the OpenVPN server is running.

### Step 6 — Add the Route

Routes must be added to the server so that your team’s clients know which traffic to route to the VPN Server.

You can determine the proper subnet by returning to your list of EC2 instances, clicking on a target instance and identifying the Private IP.

{{< figure src="/posts/vpn-2022/identify-private-ip.png" alt="Identify Private IP" align="center" attr="Image credit: Author" >}}

Your network will be the first 2 parts of the Private IP appended with zeros, e.g. _172.31.0.0_

On the VPN Server edit _/etc/openvpn/server/server.conf_ and add something like the following:

```
push "route 172.31.0.0 255.255.0.0"
```

Then restart the VPN Server with:

```
$ systemctl restart openvpn-server@server.service
```

### Step 7 — Grant Access to Your VPN

Note: We assume that you are still SSH’d into the VPN and logged in as root.

Run the following command and be sure to replace _client_ below with a unique name for your user/client.

```
$ /home/ubuntu/openvpn-server-vagrant/add-client.sh client
```

You’ll then find a configuration file at

```
~/client-configs/files/client-name.ovpn
```

You will want to provide this file to the individual on your team who will be connecting to your VPN. SCP is handy for downloading this .ovpn file from your VPN Server.

Your team can use one of various VPN clients such as [Tunnelblick](https://tunnelblick.net/) (OS X) and [OpenVPN](https://openvpn.net/community-downloads/) (Linux, iOS, Android and Windows). After installing one of these clients they should be able to set up the VPN config just by double clicking on the .ovpn file.

Note: once connected to the VPN, your users will want to use the Private IPs of your EC2 instances. You’ll probably want to use Route 53 to create subdomain records that route to the Private IPs.

### Step 8 — Revoke Access to Your VPN

Note: We assume that you are still SSH’d into the VPN and logged in as root.

Run the following command and be sure to replace _client_ below with a unique name for your user/client.

```
$ /home/ubuntu/openvpn-server-vagrant/revoke-full.sh client
```

### Troubleshooting

If your VPN client reports a _TLS handshake failed_ error then this is most likely because your VPN security group (Step 1) is incorrect. Make sure that you have the correct ports and protocols specified — a common problem is not specifying UDP for port 1194.

{{< disqus >}}