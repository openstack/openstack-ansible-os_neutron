=============================
Scenario - Using Open vSwitch
=============================

Overview
~~~~~~~~

Operators can choose to utilize Open vSwitch instead of Linux Bridges for the
neutron ML2 agent. This offers different capabilities and integration points
with neutron. This document outlines how to set it up in your environment.

Recommended reading
~~~~~~~~~~~~~~~~~~~

We recommend that you read the following documents before proceeding:

 * Neutron documentation on Open vSwitch OpenStack deployments:
   `<https://docs.openstack.org/liberty/networking-guide/scenario-classic-ovs.html>`_
 * Blog post on how OpenStack-Ansible works with Open vSwitch:
   `<https://medium.com/@travistruman/configuring-openstack-ansible-for-open-vswitch-b7e70e26009d>`_

Prerequisites
~~~~~~~~~~~~~

All compute nodes must have bridges configured:

- ``br-mgmt``
- ``br-vlan`` (optional - used for vlan networks)
- ``br-vxlan`` (optional - used for vxlan tenant networks)
- ``br-storage`` (optional - used for certain storage devices)

For more information see:
`<http://docs.openstack.org/project-deploy-guide/openstack-ansible/newton/targethosts-networkconfig.html>`_

These bridges may be configured as either a Linux Bridge (which would connect
to the Open vSwitch controlled by neutron) or as an Open vSwitch.

Configuring bridges (Linux Bridge)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following is an example of how to configure a bridge (example: ``br-mgmt``)
with a Linux Bridge on Ubuntu 16.04 LTS:

``/etc/network/interfaces``

.. code-block:: shell-session

    auto lo
    iface lo inet loopback

    # Management network
    auto eth0
    iface eth0 inet manual

    # VLAN network
    auto eth1
    iface eth1 inet manual

    source /etc/network/interfaces.d/*.cfg

``/etc/network/interfaces.d/br-mgmt.cfg``

.. code-block:: shell-session

    # OpenStack Management network bridge
    auto br-mgmt
    iface br-mgmt inet static
      bridge_stp off
      bridge_waitport 0
      bridge_fd 0
      bridge_ports eth0
      address MANAGEMENT_NETWORK_IP
      netmask 255.255.255.0

One ``br-<type>.cfg`` is required for each bridge. VLAN interfaces can be used
to back the ``br-<type>`` bridges if there are limited physical adapters on the
system.

Configuring bridges (Open vSwitch)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Another configuration method routes everything with Open vSwitch. The bridge
(example: ``br-mgmt``) can be an Open vSwitch itself.

The following is an example of how to configure a bridge (example: ``br-mgmt``)
with Open vSwitch on Ubuntu 16.04 LTS: *

``/etc/network/interfaces``

.. code-block:: shell-session

    auto lo
    iface lo inet loopback

    source /etc/network/interfaces.d/*.cfg

    # Management network
    allow-br-mgmt eth0
    iface eth0 inet manual
      ovs_bridge br-mgmt
      ovs_type OVSPort

    # VLAN network
    allow-br-vlan eth1
    iface eth1 inet manual
      ovs_bridge br-vlan
      ovs_type OVSPort

``/etc/network/interfaces.d/br-mgmt.cfg``

.. code-block:: shell-session

    # OpenStack Management network bridge
    auto br-mgmt
    allow-ovs br-mgmt
    iface br-mgmt inet static
      address MANAGEMENT_NETWORK_IP
      netmask 255.255.255.0
      ovs_type OVSBridge
      ovs_ports eth0

One ``br-<type>.cfg`` is required for each bridge. VLAN interfaces can be used
to back the ``br-<type>`` bridges if there are limited physical adapters on the
system.

**Warning**: There is a bug in Ubuntu 16.04 LTS where the Open vSwitch service
won't start properly when using systemd. The bug and workaround are discussed
here: `<http://www.opencloudblog.com/?p=240>`_


OpenStack-Ansible user variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set the following user variables in your
``/etc/openstack_deploy/user_variables.yml``: *

.. code-block:: yaml

  # Ensure the openvswitch kernel module is loaded
  openstack_host_specific_kernel_modules:
    - name: "openvswitch"
      pattern: "CONFIG_OPENVSWITCH="
      group: "network_hosts"

  ### neutron specific config
  neutron_plugin_type: ml2.ovs

  neutron_ml2_drivers_type: "flat,vlan"

  # Typically this would be defined by the os-neutron-install
  # playbook. The provider_networks library would parse the
  # provider_networks list in openstack_user_config.yml and
  # generate the values of network_types, network_vlan_ranges
  # and network_mappings. network_mappings would have a
  # different value for each host in the inventory based on
  # whether or not the host was metal (typically a compute host)
  # or a container (typically a neutron agent container)
  #
  # When using Open vSwitch, we override it to take into account
  # the Open vSwitch bridge we are going to define outside of
  # OpenStack-Ansible plays
  neutron_provider_networks:
    network_flat_networks: "*"
    network_types: "vlan"
    network_vlan_ranges: "physnet1:102:199"
    network_mappings: "physnet1:br-provider"

Customization is needed to support additional network types such as vxlan,
GRE or Geneve. Refer to the `neutron agent configuration
<https://docs.openstack.org/neutron/latest/configuration/#configuration-reference>`_ for
more information on these attributes.
