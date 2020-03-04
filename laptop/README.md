# Remote Machine Ansible Scripts

## Setting up the remote machines

The remote machines have to have SSH setup and connected to teh network to start the configuration. It is recommended to start with the [Kali Linux Mini iso](http://http.kali.org/kali/dists/kali-rolling/main/installer-amd64/current/images/netboot/mini.iso). 

Once the base operating system is installed, make sure that SSH is working and that the root account is allowed to sign in. Using the server as a starting point, use `ssh-copy-id` to copy the auto generated SSH key to the remote machine. This is required because Ansible does not like using password based authentication. 

Once SSH is setup with Public Key Authentication from the server to the remote machine, then continue to the **Using this playbook** section.

## Using this playbook
This ansible playbook will be used from the server. To run this playbook (on the server), use the following command:

```
ansible-playbook site.yml -K -i inventory.yml
```

