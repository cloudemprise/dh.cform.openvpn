# cfn-ovpn-cli

> A virtual private network application in the cloud.

<p align="center">
  <img src="./docs/images/cfn-ovpn-cli-sys-overview.png">
</p>

A hardened and highly available, multi-client, dual protocol, VPN appliance, accompanied by an isolated Public Key Infrastructure framework, orchestrated in Cloudformation via the AWS command line interface.

highly available
fault tolerant
scalable
elastic
cost efficient

[![Linux](https://img.shields.io/badge/OS-Linux-blue?logo=linux)](https://github.com/cloudemprise/cfn-ovpn-cli)
![Bash](https://img.shields.io/badge/Bash->=v4.0-green?logo=GNU%20bash)
[![jq](https://img.shields.io/badge/jq-v1.6-green.svg)](https://github.com/stedolan/jq)
[![awscli](https://img.shields.io/badge/awscli->=v2.0-green.svg)](https://github.com/aws/aws-cli)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

<details>  
  <summary><b>Click to View Sample CLI Output:</b></summary>

  <p align="center">
    <img src="./docs/images/cfn-ovpn-cli-terminal-sample.svg">
  </p>

</details><br/>

**Full Demo CLI Output can be found Here**

## Prerequisites

- aws account.
- ssh key in stack build region.
- route 53 hosted zone.
- jq version 1.6
- awscli version 2
- bash > version 4

Table of Contents
=================

- [Introduction](#introduction)
- [Cloudformation](#cloudformation)
- [OpenVPN](#openvpn)
- [Pulic Key Infrastructure](#public-key-infrastructure)
- [Network Security](#network-security)
  * [AWS Network ACLs](#aws-network-acls)
  * [AWS Security Groups](#aws-security-groups)
  * [The Linux Kernel Packet Filter](#the-linux-kernel-packet-filter)
  * [Intrusion Prevention](#intrusion-prevention)
  * [sshd Hardening](#sshd-hardening)
- [System Hardening](#system-hardening)
  * [AWS IAM Instance Roles](#aws-iam-instance-roles)
  * [AWS Instance Metadata Service](#aws-instance-metadata-service)
  * [Package Management](#package-management)
- [Telemetry](#telemetry)
  * [Amazon CloudWatch Logs](#amazon-cloudwatch-logs)
  * [rsyslog](#rsyslog)
- [Error Handling](#error-handling)
- [Conclusion](#conclusion)
- [To Do List](#to-do-list)

## Introduction

**cfn-ovpn-cli** is a shell script that creates a cloud-based Virtual Private Network (VPN) application together with a isolated Public Key Infrastructure (PKI) Certification Authority, that provides for a secure mobile Wi-Fi roaming solution. The AWS Command Line Interface (AWS CLI) is used to provision and configure various AWS Resources through an assortment of API calls and AWS Cloudformation templates.
Talk a little about bash script here.

## Cloudformation

AWS Cloudformation is a service that provisions and configures cloud resources that are declared in a template file. The template defines a collection of elements as a single unit called a Stack, simplifying the management of cloud infrastructure.

**cfn-ovpn-cli** templates compose a monolithic hierarchical tree structure of nested stacks and orchestration is achieved in a three-phase stack creation/update process that is promoted via a counter variable. The flowchart below illustrates the build process and resouce dependencies:

<details>  
  <summary>Click to View Flow Chart</summary>

  <p align="center">
    <img src="./docs/images/cfn-flowchart.png">
  </p>

</details>

## OpenVPN

OpenVPN is a popular VPN daemon that is remarkably flexible and relatively simple to setup. It is particularly suitable for small deployments and uses the TLS protocol to secure its tunnels.  It derives its cryptographic capabilities from the OpenSSL library but can also be compiled with [Mbed TLS](https://tls.mbed.org) as its cryptographic backend. This is supposed to ensure the independence of the underlying encryption libraries. 

OpenVPN operates by using a virtual network adapter as an interface between user-space and kernel-space and listens for client connections on both UDP and TCP. OpenVPN defines the concept of a control channel and a data channel, both of which are encrypted and secured separately but pass over the same protocol connection. The control channel is encrypted and secured using TLS while the data channel is encrypted using a stipulated block cipher.

**cfn-ovpn-cli** is setup in the following fashion:

**cfn-ovpn-cli** uses the following parameters:

|   | Control Channel | Data Channel
| :----: |:---: | :---: 
| Encryption | secp521r1 | AES-256-GCM
| Authentication | sha512 | sha512

the client presents a list of supported cipher suites (ciphers and hash functions).
https://en.wikipedia.org/wiki/Transport_Layer_Security

cipher AES-256-GCM
auth SHA512
tls-version-min 1.2
tls-crypt hmac-sig.key
tls-cipher TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384

ephemeral elliptic-curve Diffie–Hellman (TLS_ECDHE)
Only TLS_DHE and TLS_ECDHE provide forward secrecy. 

key size = robustness of the security
encryption strength is directly related to the key size

Elliptic Curve Digital Signature Algorithm (ECDSA)
https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm


Section on how it is installed and configured and manintained Bla bla
**cfn-ovpn-cli** configures OpenVPN as such: bla bla
It is used in both protocols methods and configures four clients, i.e. bla bla


## Public Key Infrastructure

A secure VPN requires authentication and this here involves two components:

1. Client/Server Authentication. This ensures that the server and clients are indeed communicating with authorized known entities and not some spoofed fake user/host.

2. A method of hashing each data packet within the system is also established. By authenticating each data packet, the system avoids wasting cpu cycles on decrypting packets that don't meet the authentication rules. Thus preventing many types of attack vectors.

A system that uses key-based authentication requires a Public Key Infrastructure (PKI). Simply put, a PKI consists of the following:

* A public master Certificate Authority (CA) certificate and a private key.
* A separate public certificate and private key pair for the server.
* A separate public certificate and private key pair for each client.

Conveniently, a awesome utility exist just for the purposes of making PKI management really easy, i.e. [Easy-RSA](https://github.com/OpenVPN/easy-rsa). Easy-RSA is a framework for managing X.509 PKI. It is based around the concept of a trusted root signing authority and the backend is comprised of the OpenSSL cryptographic library.

**cfn-ovpn-cli** builds two interrelated PKIs on hardened virtual linux servers within a VPC in the AWS Cloud. A trusted root CA is created within the isolated private subnet, only accessible via a Bastion Host and used exclusively to sign certificate requests. A second PKI is created on the OpenVPN application server itself which resides within the public subnet of the VPC. Here the server as well as the client certificates and private keys are generated. Requests and signed certificates are intelligently exchanged between these two systems by way of Cloudformation Stack Updates. This is described in more detail in the Cloudformation section below but suffice to say, hinges around the calling of the ec2 create-image API. The CA is further constrained by a passphrase that is securely stored and retrived via the AWS System Manager Parameter Store secrets management protocol. The elliptical curve secp521r1 key exchange cipher was chosen for smaller key size equivalence and faster execution performance.

export RANDFILE=/tmp/.rnd

## Network Security

Defense in depth is adopted to provide redundancy.

### AWS Network ACLs

Amazon VPC offers a network access control list (NACL) element that provides for an optional layer of security in the form of a stateless firewall, acting at the subnet layer. 

**cfn-ovpn-cli** assembles an ordered set of rules for its public as well as its private subnets. These two NACLs grant only the necessary inbound source or outbound destination routes for their respective privileged or unprivileged ports. All other traffic is denied from even entering the virtual network itself. 

### AWS Security Groups

While NACLs operate at the network subnet level, security groups act at the instance level. An instane is a unit of compute capacity, eg. virtual machine, container or database etc. Conceptually, a secutity group can be thought of as a virtual statefull firewall, surrounding some computational unit, i.e. instance. 

A collection of unordered rules decide whether to allow traffic to reach the computation unit or not. An interesting feature of security groups is that they can reference other security groups as their end-point, exhibiting object like characteristics that simplifying rule creation and management.

**cfn-ovpn-cli** constructs two security groups, a Public Security Group for instances residing on public subnets and similarly, a Private Security Group for instances residing on private subnets. Restrictive rules are applied to ensure only expected traffic is allowed to pass through to the operating system.

### The Linux Kernel Packet Filter

iptables is a user-space utility program that can configure the Linux kernel packet filtering framework as well as NAT and other packet mangling tasks. Various modules can be side-loaded to extend its complex filtering capabilities. 

During Stage-2 of the Cloudformation Stack build process, **cfn-ovpn-cli** installs and configures iptables in a statefull posture within the Public and Private Virtual Linux Servers. Strict rules delineate distinct traffic domains for localhost services, the AWS instance metadata service (IMDS) and forwarded NAT VPN client traffic. As a precautionary measure VPN clients are denied routes to the local LAN.

* input pings are open to the world but rate limited
* input ssh is also rate limited
* ntp only allow to IMDS

### Intrusion Prevention

fail2ban is an intrusion prevention system that safeguards against network brute-force attacks. It operates in conjunction with 
the Linux kernel packet filter, i.e. iptables and monitors log files for selected entries to block offending hosts. **cfn-ovpn-cli** configures a standard filter for sshd on port 22 in aggressive mode.

To Do : OpenVPN filter.

### sshd Hardening

**cfn-ovpn-cli** not only utilizes an external-facing public EC2 instance as a VPN server but also as a Bastion Host, specifically providing access to the PKI residing within its private network. Because of its exposure to potential attacks, the onus is to minimize any chance of penetration, increase the defense of the system and reduce any potential risks from zero day exploits.

Pertaining to the the golden image _Amazon Linux 2_, only non-default settings are discussed here.

- The following straightforward changes are made to the default configuration:

| keyword-argument | Description |
| :---- | :--- |
| `X11Forwarding no` | X11 forwarding is an insecure feature and thus disabled. |
| `PermitRootLogin no` | Root login is denied. |
| `AllowUsers ec2-user` | The default linux system user account enabled only. |
| `LogLevel VERBOSE` | Comprehensive logging enabled. |

- The following settings combined, explicitly deny all forms of authentication except for Public Key:

| keyword-argument | Description |
| :---- | :--- |
| `AuthenticationMethods publickey` | Authentication method required for access granted. |
| `PubkeyAuthentication yes` | Public key authentication allowed |

- Together the following settings bring about an auto-disconnect feature for idle or inactive ssh sessions:

| keyword-argument | Description |
| :---- | :--- |
| `ClientAliveCountMax 0` | Disable keep-alive pings. |
| `ClientAliveInterval 600` | 10 min. inactive timeout interval. |

- Automatic security updates are routinely applied as discussed in the section on [Package Management](#package-management).
- Pursuant to [RFC8270](https://tools.ietf.org/html/rfc8270) and other endorsed infosec [recommendations](https://infosec.mozilla.org/guidelines/openssh), Diffie-Hellman moduli of less than 3072 bits are not recommended. As such short moduli are deactivated. 
- **cfn-ovpn-cli** does not require sftp support, therefore it is disabled.

## System Hardening

* Encrypted EBS-volume

* Chosen OS _Amazon Linux 2_ / Centos 7 knock-off / Custom Image

* chrony only local (iptables allows only ntp to IMDS)

Ancillary: As is discussed else where, Security Groups, iptables also offer lines of defense.
Include bash function aliases to authorize my ip.

### IAM Instance Roles

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

### Instance Metadata Service

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

### Package Management

Keeping **cfn-ovpn-cli** up-to-date is important because it improves the overall stabiltity of the system by ensuring that critical patches to security holes are applied in a timely manner. It also ensures that bugs are fixed and outdated features are regularly removed, making the nature of the offered service more beneficial.

By default, _Amazon Linux 2_ instances launch with the following repositories enabled:

| Repository | Description |
| :----: | :---: |
| `amzn2-core` | Amazon Linux 2 core repo |
| `amzn2extra-docker` | Amazon Extras repo for docker |

`amzn2extra-docker` is not required and thus disabled.

The Extra Packages for Enterprise Linux (`epel`) repository is a non-standard but popular archive managed by the Fedora Project. It is compatible with _Amazon Linux 2_ and contains the various auxillary packages required by **cfn-ovpn-cli**.

Due to packages perhaps being held by more than one repository, `yum-plugin-priorities` is used to enforce ordered protection between conflicting repositories. `epel` is thus set at a lower priority to `amzn2-core`.

The `yum-cron` service is configured to automatically keep the system up-to-date.

## Telemetry

The _Amazon Linux 2_ golden image uses _systemd-journald_ as well as _rsyslog_ for event logging. Important log streams are configured, collated and dispatched to _Amazon CloudWatch Logs_ and the _systemd_ journal is made persistent across reboots. To restrict the volume of local log data, the utility _logrotate_ is configured to compress and rotate the relevant streams.

### Amazon CloudWatch Logs

Amazon CloudWatch Logs is a web service that provides aggregation and analysis tools for distributed message logging systems. **cfn-ovpn-cli** utilizes _Cloudformation_ to install and configure a _Unified CloudWatch Agent_ that facilitates the collection and retention of operational log events. The Agent is authorized by the IAM _CloudWatchAgentServerPolicy_ managed policy.

### rsyslog

For improved parsing and assisted analysis, **cfn-ovpn-cli** configures custom log-streams for the linux kernel packet filter (`iptables`) as well the for both the VPN daemon protocol activities. These are diverted to a separate log facility for forwarding to _Amazon Cloudwatch Logs_.

## Error Handling

Talk about health-checks etc.


## Conclusion

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

## To Do List

OpenVPN client scripts

Move iptables logs with Cloudwatch agent.

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.