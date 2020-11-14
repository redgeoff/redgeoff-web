+++
title = "Running a Free VPN Server on AWS"
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

{{< img src="/posts/vpn/network-with-vpn.png" alt="Network with VPN" caption="Source: http://computer.howstuffworks.com/vpn3.htm" align="center" >}}

AWS has an awesome firewall built into its core services which can easily be used to make sure that only certain ports are open to the outside world. One extra step that we can take is to run a VPN Server that serves as the gateway to our protected EC2 instances. We can then shutdown direct SSH access to our EC2 instances and also have the freedom to block access to our entire network just by revoking access via our VPN Server. The later is very useful if you need to revoke access for a former employee.

The following tutorial will take you through the steps of setting up an EC2 instance that will run the OpenVPN Server. It will then cover how to grant and revoke access through the VPN Server.

### Step 1— Create the VPN Security Group

Overview: security groups allow your servers to communicate with each other in a private cloud while exposing specific ports to the world. We are going to create a security group to allow VPN access to our VPN Server. We will assume that all your other EC2 instances are members of the default security group and that the default security group does not allow access from the outside world.

Log in at [https://aws.amazon.com](https://aws.amazon.com), type EC2 in the search box and click on the target to go to the EC2 Dashboard.

From the EC2 dashboard, click _Security Groups_

{{< img src="/posts/vpn/security-group.png" alt="Security Groups" align="center" >}}

