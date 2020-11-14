+++
title = "Running a Free VPN Server on AWS"
description = "Run OpenVPN on an AWS EC2 instance"
images = ["/posts/vpn/network-with-vpn.png"]
date = 2017-07-24T07:39:02-08:00
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

{{< img src="/posts/vpn/network-with-vpn.png" alt="Network with VPN" align="center" >}}

This tutorial will take you through the steps of setting up an EC2 instance that will run the OpenVPN Server. It will then cover how to grant and revoke access through the VPN Server.

AWS has an awesome firewall built into its core services which can easily be used to make sure that only certain ports are open to the outside world. One extra step that we can take is to run a VPN Server that serves as the gateway to our protected EC2 instances. We can then shutdown direct SSH access to our EC2 instances and also have the freedom to block access to our entire network just by revoking access via our VPN Server. The later is very useful if you need to revoke access for a former employee.

### Step 1— Create the VPN Security Group

Overview: security groups allow your servers to communicate with each other in a private cloud while exposing specific ports to the world. We are going to create a security group to allow VPN access to our VPN Server. We will assume that all your other EC2 instances are members of the default security group and that the default security group does not allow access from the outside world.

Log in at [https://aws.amazon.com](https://aws.amazon.com), type EC2 in the search box and click on the target to go to the EC2 Dashboard.

From the EC2 dashboard, click _Security Groups_:

{{< img src="/posts/vpn/security-group.png" alt="Security Groups" align="center" >}}

Click Create _Security Group_:

{{< img src="/posts/vpn/create-security-group.png" alt="Create Security Group" align="center" >}}

Enter a name and description of vpn and specify inbound rules on ports 22, 443, 943 and 1194. Note: the protocol for port 1194 is UDP.

{{< img src="/posts/vpn/ports.png" alt="Ports" align="center" >}}

Note: if the IP addresses that your team uses are static then you can add yet another layer of security by specifying that IP address range in the Source of your rules. However, you’ll want to leave the Source blank if you want your team to be able to connect from different IPs as they may be working from a hotel, home, cafe, etc…

### Step 2 — Create the EC2 Instance

Return to the EC2 Dashboard and then click _Launch Instance_:

{{< img src="/posts/vpn/create-instance.png" alt="Create EC2 Instance" align="center" >}}

Select Ubuntu (you can of course select almost any other OS that runs OpenVPN, but this tutorial is tailored for Ubuntu)

{{< img src="/posts/vpn/ubuntu.png" alt="Ubuntu 16 AMI" align="center" >}}

Select t2.nano and click _Review and Launch_

{{< img src="/posts/vpn/t2-nano.png" alt="t2.nano" align="center" >}}

On the next screen, click _Edit security groups_

{{< img src="/posts/vpn/security-groups.png" alt="Edit security groups" align="center" >}}

Select the _vpn_ and _default security groups_ and click _Review and Launch_

{{< img src="/posts/vpn/review-and-launch.png" alt="Review and Launch" align="center" >}}

Click _Launch_, choose your key pair and then click _Launch Instances_

### Step 3 — Disable Source/Destination Check

From the list of instances, select the VPN instance and then Networking->Change Source/Dest. Check from the drop down menu. Then click Yes, Disable. This is needed as otherwise, your VPN server will not be able to connect to your other EC2 instances.

{{< img src="/posts/vpn/disable-check.png" alt="Disable Source/Destination Check" align="center" >}}

### Step 4— Create an Elastic IP Address

Overview: when an EC2 instance is stopped and restarted, the Public IP address changes. We want the IP address of our VPN Server to remain static so we’ll use an Elastic IP Address.

From the E2c Dashboard, select _Elastic IPs_

{{< img src="/posts/vpn/elastic-ips.png" alt="Elastic IPs" align="center" >}}

Click _Allocate new address_

{{< img src="/posts/vpn/allocate-new-address.png" alt="Allocate New Address" align="center" >}}

Click _Allocate_ and then _Close_.

Make a note of your Elastic IP address as this will be the Public IP Address of your VPN Server.

Then select the Elastic IP and click _Associate address_ from the drop down menu.

{{< img src="/posts/vpn/associate-address.png" alt="Associate Address" align="center" >}}

Select the EC2 instance you just created and click _Associate_.

{{< img src="/posts/vpn/associate-private-ip.png" alt="Associate Private IP" align="center" >}}

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

Update Ubuntu and install OpenVPN. Note: you will be prompted twice and when you do, select `Keep the local version currently installed`

```
$ /home/ubuntu/openvpn-server-vagrant/ubuntu.sh
$ /home/ubuntu/openvpn-server-vagrant/openvpn.sh
```

At this point, the OpenVPN Server is running.

### Step 6 — Add the Route

Routes must be added to the server so that your team’s clients know which traffic to route to the VPN Server.

You can determine the proper subnet by returning to your list of EC2 instances, clicking on a target instance and identifying the Private IP.

{{< img src="/posts/vpn/identify-private-ip.png" alt="Identify Private IP" align="center" >}}

Your network will be the first 2 parts of the Private IP appended with zeros, e.g. 172.31.0.0

On the VPN Server edit _/etc/openvpn/server.conf_ and add something like the following:

```
push "route 172.31.0.0 255.255.0.0"
```

Then restart the VPN Server with:

```
$ systemctl restart openvpn@server
```

### Step 7— Grant Access to Your VPN

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

### Step 8— Revoke Access to Your VPN

Note: We assume that you are still SSH’d into the VPN and logged in as root.

Run the following command and be sure to replace _client_ below with a unique name for your user/client.

```
$ /home/ubuntu/openvpn-server-vagrant/revoke-full.sh client
```

### Troubleshooting

If your VPN client reports a _TLS handshake failed_ error then this is most likely because your VPN security group (Step 1) is incorrect. Make sure that you have the correct ports and protocols specified — a common problem is not specifying UDP for port 1194.

{{< disqus >}}