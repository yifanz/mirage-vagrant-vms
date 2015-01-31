# Mirage Virtualbox VMs via Vagrant

## Setup

### Clone this repo

    $ git clone https://github.com/mirage/mirage-vagrant-vms.git
    $ cd mirage-vagrant-vms

### Installing Vagrant

First, install [Vagrant][]. On OSX I use [homebrew][] so I do this as follows:

    $ brew tap phinze/cask
    $ brew install brew-cask
    $ brew cask install vagrant
    $ vagrant --version
    Vagrant 1.4.3

[homebrew]: http://brew.sh/
[vagrant]: http://vagrantup.com/

### Installing packer

Download the [appropriate package](http://www.packer.io/downloads.html) and
install it.

On OSX,

    $ brew tap homebrew/binary
    $ brew install packer

### Before building the VM

Create a directory that will be shared between the host and guest systems.

    $ mkdir /tmp/mirage-vagrant-vms

On the guest system this will be shared as `/host`. To use a different directory
for host/guest/both, then modify `Vagrantfile.template`.

## Building the VM

    $ packer build templates/ubuntu-14.10-amd64.json
    $ cp Vagrantfile.ubuntu-14.10-xen Vagrantfile

Finally, bring up a VM from the box and login; the first time this creates lots
of output as the VM is created, initialised and provisioned. Administrator
privilege is required to create the NFS mounts on the host so that the host
filesystem can be shared with dom0 in the VM (the VirtualBox guest additions do
not work with dom0).

    $ vagrant up
    ...
    $ vagrant ssh
    Linux wheezy-xen 3.2.0-4-amd64 #1 SMP Debian 3.2.54-2 x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Sat Feb 15 13:18:04 2014 from 10.0.2.2
    Linux debian-7 3.2.0-4-amd64 #1 SMP Debian 3.2.54-2 x86_64 GNU/Linux
    Sat Feb 15 20:51:26 UTC 2014

    : vagrant@wheezy-xen:~$;

And that's it -- subsequently, `vagrant halt` will stop the VM (or the usual
`shutdown -h now` when logged into it), `vagrant up` will restart it, and
`vagrant ssh` to login. The VM is accessible from the host at the specified
address (by default, `192.168.77.2`).

If you want to customise the box, I suggest looking at the `Vagrantfile.template` plus
the scripts in `provisioning/`.

If when logging in the first time after provisioning you find that the shared
filesystem is not accessible (by default at `/host`), logout, `vagrant halt` and
`vagrant up`.

## Networking

At the risk of this becoming a "general" Xen networking tutorial (as if such a
thing were possible)...


             virtualbox

    [ dom0       ]    [ mirage-domU ]
    [ 100.64.0.2 ]    [ 100.64.0.3  ]
                      [ vifN.0      ]
         |                | 
         +---[ xenbr0 ]---+
                 |
                 |
              [ eth2 ]
                 |
                 |


### Useful commands

On host:

    vboxmanage list hostonlyifs
    vboxmanage list dhcpservers

On dom0:

    sudo xm network-list mort-www


### /etc/network/interfaces

network interface configuration. eth0-N, xenbr0

    auto xenbr0
    iface xenbr0 inet dhcp
          bridge_ports eth2
          bridge_stp off
          bridge_waitport 0
          bridge_fd 0

    #VAGRANT-BEGIN
    # The contents below are automatically generated by Vagrant. Do not modify.
    auto eth1
    iface eth1 inet static
          address 172.16.0.2
          netmask 255.255.255.0
    #VAGRANT-END

    #VAGRANT-BEGIN
    # The contents below are automatically generated by Vagrant. Do not modify.
    auto eth2
    iface eth2 inet static
          address 100.64.0.2     <*>
          netmask 255.255.255.0  <*>
    #VAGRANT-END

Lines marked <*> move to the `xenbr0` config from the `eth2` config, and the `eth2` config marked as `manual` not `static`. This makes connection from `dom0` to `mirage-domU` work.

### /etc/dhcp/dhcpd.conf

DHCP server config. subnets from which addresses should be responded with.

    subnet 100.64.0.0 netmask 255.255.255.0 {
      range 100.64.0.2 100.64.0.254;
      # option routers rtr-239-0-1.example.org, rtr-239-0-2.example.org;
    }
