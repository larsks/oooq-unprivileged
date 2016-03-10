# Running TripleO-Quickstart as an unprivileged user

It is possible to run tripleo-quickstart as an unprivileged user,
although you will need to make some one-time changes to your system to
provide the necessary infrastructure.

## Create libvirt networks

You will need to create two libvirt networks (or, more accurately, one
libvirt network and a bridge).  The first is we refer to as the
"external" network; if you have the standard libvirt "default" network
available, that's fine, otherwise you can create an appropriate
network like this:

    # cat <<EOF | virsh net-define /dev/stdin
    <network>
      <name>external</name>
      <forward mode='nat'>
        <nat>
          <port start='1024' end='65535'/>
        </nat>
      </forward>
      <bridge name='brext' stp='on' delay='0'/>
      <ip address='192.168.23.1' netmask='255.255.255.0'>
        <dhcp>
          <range start='192.168.23.2' end='192.168.23.254'/>
        </dhcp>
      </ip>
    </network>
    EOF
    # virsh net-start external
    # virsh net-autostart external

You will need a bridge device that will be used as an "internal"
network for provisioning the overcloud machines (and as a general
purpose network for the overcloud):

    # cat <<EOF | virsh net-define /dev/stdin
    <network>
      <name>overcloud</name>
      <bridge name='brovc' stp='off' delay='0'/>
    </network>
    EOF
    # virsh net-start overcloud
    # virsh net-autostart overcloud

## Whitelist bridge devices

In order to make use of these networks as an unprivileged user, they
must be whitelisted in the configuration for the QEMU bridge helper.
Edit `/etc/qemu/bridge.conf` (this may be `/etc/qemu-kvm/bridge.conf`
on your system) and add:

    allow brext
    allow brovc

## Run the playbooks

Now, as a non-root user, you can place the following in
`playbook.yml`:

    - hosts: localhost
      roles:
        - libvirt/setup

    - hosts: undercloud
      roles:
        - tripleo

And the following in `config.yml` (these values must, of course,
match the libvirt networks we created earlier, so if you're using the
libvirt `default` network you will need different values here):

    networks:
      - name: external
        bridge: brext
        address: 192.168.23.1
        netmask: 255.255.255.0

      - name: overcloud
        bridge: brovc

And run it like this:

    ansible-playbook playbook.yml -e @config.yml

This will set up the default environment, which requires at least 32GB
of RAM.  If you want to deploy a smaller environment, consider adding
the following to your `config.yml`:

    undercloud_memory: 8000

    control_memory: 6000
    compute_memory: 2000

    overcloud_nodes:
      - name: control_0
        flavor: control
      - name: compute_0
        flavor: compute

This will deploy a single controller and a single compute node, and
requires around 16GB of memory.
