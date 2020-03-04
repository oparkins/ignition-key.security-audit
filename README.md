# ignition-key.security-audit
Ansible Scripts to Automatically Setup a Security Audit Infrastructure 

Created for New Mexico Institute of Mining and Technology's CyberCorp Scholarship For Service Program.

## About

This repository contains [Ansible](https://www.ansible.com/) scripts to
configure a server and multiple remote machines for security audits. 

The Ansible scripts will setup a server to use SSH certificates for
authentication purposes. SSH certificates are used because they allow for easier
key management when working with a group of people. See the `SSH Key Management`
Section for information on how to allow users to access the server and remote
machines.

The Ansible scripts will also setup OpenVAS in a master/slave architecture. This
allows for remote machines to be managed easier and for results to automatically
be collected without user intervention. 

## Server Setup

A server needs to have at least 2 CPUs, 4 GB of RAM, and 20 GBs of hard drive space. 

Once a server has been created, either physical or virtual, it needs to have an
operating system installed on it. It is currently recommended to use Ubuntu
18.04 LTS. While CentOS would have been a good choice for a server, Ubuntu
repositories have more of the packages that we require. After installing the
operating system, add your SSH key to your account on the server. Ansible works
best with SSH keys. 

Now navigate to the `ansible/server/` directory of this repository and edit the
[inventory](./ansible/server/inventory) file to include your server. An example
would be:
```
[all]
audit.example.org:2222
```

If you are using a standard SSH port, you do not need to include the `:2222`. 

**It is not currently possible to setup multiple servers at the same time. This is because the master SSH Key is copied to the same location for all servers.**

After editing the inventory file, edit the `ansible/vars.yml` file. This is what
Ansible uses to configure the infrastructure. An example is:
```
server_address: audit.example.org
server_ssh_port: 2222
audit_user: silly-shirly
ssh_base_port: 10000
ssh_ping_port: 11000
openvas_base_port: 12000
```

You shouldn't need to change the base port information. The only thing that you
should change is the `audit_user`, server address, and server port. The
`audit_user` is a unique name among all audits, but is not derived from the
client's name. This allows for team members to talk about the client without
giving away to other people around. It is also used for username authentication
later.

There is a password hash listed in the laptop configuration files. It is the user account to sign in locally on the machine. It is recommend that the hash is changed to suit your organizational needs.


After editing the configuration files, run the following command inside the
`ansible/server/` directory:

```
ansible-playbook -i inventory site.yml -K
```

The `-K` makes Ansible ask for your sudo password so that it can do required
*sudo* operations. If you need to use a different user account name than what is
on your local machine, use the `-u` flag. Google is your friend for other
issues.

After the playbook is ran, you will not be able to access the server through SSH
unless you have the master SSH public key. It is recommended to copy this to
your `.ssh` folder. Be sure to change the command below to reflect your system:
```
cp <PATH TO SECURITY-AUDIT REPO>/security-audit/ansible/server/id_rsa-cert.pub ~/.ssh/id_rsa-cert-<SERVER NAME>.pub
```

Now, add an entry into your `~/.ssh/config` file. If the file doesn't exist,
create it. The content that needs to be added is:

```
Host <SERVER NAME> <CLIENT NAME>
    Hostname <SERVER ADDRESS>
    User root
    CertificateFile ~/.ssh/id_rsa-cert-<SERVER NAME>.pub
```

This will allow you to use either `<SERVER NAME>` or `<CLIENT NAME>` in your ssh commands to connect to the server like:
```
ssh <SERVER NAME>
```

Please note that your certificate that you were given is a master key to all
machines that will be used in this infrastructure. **DO NOT SHARE WITH ANYONE ELSE**

That certificate allows root access on all machines. You will generate other
certificates for other team members to access the infrastructure without root
privileges.

For more information on what happens to the servers, view the `README.md` file
inside the `ansible/server` directory.

## Remote Machine Setup 

After the server is configured, there will be a directory in the root folder
that contains Ansible scripts to setup the remote machines. The scripts can
setup multiple remote machines at the same time. The scripts will setup OpenVAS
scanner on the remote machines and setup SSH tunnels back to the server
for secure communications.

There will be three ports that will open on the server per laptop. One will be a
tunnel back to the ssh port on the laptop, one will be a ping/keep alive port
that autossh uses, and the last port is for OpenVAS slave connections. 

All the remote machines will recognize the CA that was setup on the server. This
makes key management a simpler problem because nothing has to be transfered to
the remote machines. Instead, the team members will have to get their public key
signed by the CA which will tell the laptops to allow them to log in. 

To setup the laptops, first install Kali Linux Mini.iso onto them. You can
select the minimal install. 

Then setup the network so that the server can SSH directly to the remote
machines. Either the remote machines have to be connected to the same network,
or SSH tunnels can be used to configure the remove machines. This document will
not detail how to connect the laptops to the network - that is your
responsibility to get the server the ability to SSH to the laptops. 

Once the network is setup properly, transfer the server's public key to the
laptops as root:
```
ssh-copy-id <user>@<remote machine address>
```

Then, go to `/root/ansible-laptop` and edit the `inventory.yml` file located
there. Add all the machines you have access to and be sure to give them a
**unique** laptop number. The number is used to generate the ports that all the
machines connect to.

After the inventory has been updated, type the following command to start the
remote machine setup process:
```
ansible-playbook -u <username with ssh access> -i inventory.yml site.yml -K
```

That will start the installation process. OpenVAS will be installed and the
slave will be added to the server automatically. 

For more information on what happens to the machines, view the `README.md` file
inside the `ansible/laptop` directory.

## SSH Certificate Management

SSH Certificates are a super useful, underused technology that makes managing
machine access a breeze for a large amount of machines. Honestly, I would use it
for a small number of machines as well because of how flexible the technology
is. 

SSH Certificates are signed public keys with extra data attached to them. The
public keys are signed by a CA that the server trusts. When the certificate is
presented to the server, the signature is verified, the expiration information
is checked, and the principles (who the certificate can sign in as) is verified.
The amazing thing is that these certificates can be set to auto expire. The
Ansible scripts are automatically set to expire after 16 weeks, or about a
semester. 

In addition, multiple principles can be listed on a certificate which allows the
certificate to access different accounts on different machines. All team member
certificates will have two principles, a login user and the audit user, to be
able to use the same certificate for the server and remote machines.
Certificates also allow for SSH keys not to be shared to enter the same machine
(this had to be done on previous audits). This allows for better accountability
of team members during these audits. 

### Adding Team Members to Audits

To add team members, first get their public key files that they will be using.
They need to generate an ssh key pair on their local machine. After they send
you their public key file, you will need to sign it with the CA on the server.
First, upload the public key to the server.

Then, use the following command to sign their key to create a certificate:
```
ssh-keygen -s /var/ca/ssh/security-audit.ca -I <THEIR USERNAME>  -n <THE AUDIT USER IN VAR FILE>,auditor <THEIR PUBLIC KEY>
```

This will create a new file called `<THEIR PUBLIC KEY>-cert.pub`. Send them this
file back. They need to put this file in their `~/.ssh` folder and add a new
`~/.ssh/config` entry.

```
Host <SERVER NAME> <CLIENT NAME>
    Hostname <SERVER ADDRESS>
    User auditor
    CertificateFile ~/.ssh/<THEIR NEW CERTIFICATE>
```

This will allow them access to the server. To get access to the audit machines,
they need to use the server as a jump host:

```
ssh localhost -p <10000 + LAPTOP NUMBER> -J <SERVER NAME>
```

This will jump through the server and go to the appropriate remote machine. Team
members can add entries in their `~/.ssh/config` file to make this process
easier.
