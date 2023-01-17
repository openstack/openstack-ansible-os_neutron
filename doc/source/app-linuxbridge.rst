=============================
Scenario - Using Linux Bridge
=============================

Overview
~~~~~~~~

Operators can choose to utilize Linux Bridges instead of Open vSwitch for the
neutron ML2 agent. This document outlines how to set it up in your environment.

.. warning::

    LinuxBridge driver is considered as experimental in Neutron and is
    discouraged for usage as of today.


Prerequisites
~~~~~~~~~~~~~

All compute nodes must have bridges configured:

- ``br-mgmt`` - Bridge is used to wire LXC containers. Can be regular interface
  for bare metal deployments
- ``br-vlan`` (optional - used for vlan networks). Can be regular interface.
- ``br-vxlan`` (optional - used for vxlan tenant networks). Can be regular
  interface.
- ``br-storage`` (optional - used for certain storage devices). It's also
  used to wire LXC containers. Can be regular interface for bare metal nodes.

For more information see:
`<https://docs.openstack.org/project-deploy-guide/openstack-ansible/newton/targethosts-networkconfig.html>`_


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

OpenStack-Ansible user variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Specify provider network definitions in your
``/etc/openstack_deploy/openstack_user_config.yml`` that define
one or more Neutron provider bridges and related configuration:

.. code-block:: yaml

  - network:
      container_bridge: "br-provider"
      container_type: "veth"
      type: "vlan"
      range: "101:200,301:400"
      net_name: "physnet1"
      network_interface: "bond1"
      group_binds:
        - neutron_linuxbridge_agent
  - network:
      container_bridge: "br-provider2"
      container_type: "veth"
      type: "vlan"
      range: "203:203,467:500"
      net_name: "physnet2"
      network_interface: "bond2"
      group_binds:
        - neutron_linuxbridge_agent

When using ``flat`` provider networks, modify the network type accordingly:

.. code-block:: yaml

  - network:
      container_bridge: "br-publicnet"
      container_type: "veth"
      type: "flat"
      net_name: "flat"
      group_binds:
        - neutron_linuxbridge_agent

Specify an overlay network definition in your
``/etc/openstack_deploy/openstack_user_config.yml`` that defines
overlay network-related configuration:

.. note::

  The bridge name should correspond to a pre-created Linux bridge.

.. code-block:: yaml

  - network:
      container_bridge: "br-vxlan"
      container_type: "veth"
      container_interface: "eth10"
      ip_from_q: "tunnel"
      type: "vxlan"
      range: "1:1000"
      net_name: "vxlan"
      group_binds:
        - neutron_linuxbridge_agent

Set the following user variables in your
``/etc/openstack_deploy/user_variables.yml``:

.. code-block:: yaml

  neutron_plugin_type: ml2.lxb

  neutron_ml2_drivers_type: "flat,vlan,vxlan"
  neutron_plugin_base:
    - router
    - metering

The overrides are instructing Ansible to deploy the LXB mechanism driver and
associated LXB components. This is done by setting ``neutron_plugin_type``
to ``ml2.lxb``.

The ``neutron_ml2_drivers_type`` override provides support for all common type
drivers supported by LXB.

The ``neutron_plugin_base`` is used to defined list of plugins that will be
enabled.

If provider network overrides are needed on a global or per-host basis,
the following format can be used in ``user_variables.yml`` or per-host
in ``openstack_user_config.yml``.

.. note::

  These overrides are not normally required when defining global provider
  networks in the ``openstack_user_config.yml`` file.

.. code-block:: yaml

  # When configuring Neutron to support vxlan tenant networks and
  # vlan provider networks the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "vxlan"
    network_vxlan_ranges: "1:1000"
    network_vlan_ranges: "physnet1:102:199"
    network_mappings: "physnet1:br-provider"
    network_interface_mappings: "br-provider:bond1"

  # When configuring Neutron to support only vlan tenant networks and
  # vlan provider networks the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "vlan"
    network_vlan_ranges: "physnet1:102:199"
    network_mappings: "physnet1:br-provider"
    network_interface_mappings: "br-provider:bond1"

  # When configuring Neutron to support multiple vlan provider networks
  # the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "vlan"
    network_vlan_ranges: "physnet1:102:199,physnet2:2000:2999"
    network_mappings: "physnet1:br-provider,physnet2:br-provider2"
    network_interface_mappings: "br-provider:bond1,br-provider2:bond2"

  # When configuring Neutron to support multiple vlan and flat provider
  # networks the configuration may resemble the following:
  neutron_provider_networks:
    network_flat_networks: "*"
    network_types: "vlan"
    network_vlan_ranges: "physnet1:102:199,physnet2:2000:2999"
    network_mappings: "physnet1:br-provider,physnet2:br-provider2"
    network_interface_mappings: "br-provider:bond1,br-provider2:bond2"

