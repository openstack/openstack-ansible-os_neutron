========================================
Scenario - Open Virtual Network (OVN)
========================================

Overview
--------

Operators can choose to utilize the Open Virtual Network (OVN) mechanism
driver (ML2/OVN) instead of ML2/LXB or ML2/OVS. This offers the possibility 
of deploying virtual networks and routers using OVN with Open vSwitch, which
replaces the agent-based models used by the legacy ML2/LXB and ML2/OVS
architectures. This document outlines how to set it up in your environment.

Recommended Reading
~~~~~~~~~~~~~~~~~~~

Since this is an extension of the basic Open vSwitch scenario, it is worth
reading that scenario to get some background. It is also recommended to be
familiar with OVN and networking-ovn projects and their configuration.

* `Scenario: Open vSwitch <app-openvswitch.html>`_
* `OVN Architecture Docs <https://www.ovn.org/en/architecture/>`_
* `OpenStack Integration with OVN <https://docs.openstack.org/networking-ovn/latest/>`_
* `OVN OpenStack Tutorial <https://docs.ovn.org/en/stable/tutorials/ovn-openstack.html>`_

Prerequisites
~~~~~~~~~~~~~

* Open vSwitch >= 2.17.0

Inventory Architecture
----------------------

While OVN itself supports many different configurations, Neutron and ``networking-ovn`` leverage
specific functionality to provide virtual routing capabilities to an OpenStack-based Cloud.

OpenStack-Ansible separates OVN-related services and functions into three groups:

- neutron_ovn_northd
- neutron_ovn_controller
- neutron_ovn_gateway

The ``neutron_ovn_northd`` group is used to specify which host(s) will contain
the OVN northd daemon responsible for translating the high-level OVN
configuration into logical configuration consumable by daemons such as
``ovn-controller``. In addition, these nodes host the OVN Northbound and
OVN Southbound databases the ``ovsdb-server`` services. Members of this group
are typically the controller nodes hosting the Neutron APIs (``neutron-server``).

The ``neutron_ovn_controller`` group is used to specify which host(s) will run
the local ``ovn-controller`` daemon for OVN, which registers the local chassis
and VIFs to the OVN Southbound database and converts logical flows to
physical flows on the local hypervisor node. Members of this group are
typically the compute nodes hosting virtual machine instances (``nova-compute``).

The ``neutron_ovn_gateway`` group is used to specify which hosts are eligible to
act as an OVN Gateway Chassis, which is a node running ``ovn-controller`` that
is capable of providing external (north/south) connectivity to the tenant traffic.
This is essentially a node(s) capable of hosting the logical router performing
SNAT and DNAT (Floating IP) translations. East/West traffic flow is not limited
to a gateway chassis and is performed between an OVN chassis nodes.

When planning out your architecture, it is important to determine early if you
want to centralize OVN gateway chassis functions to a subset of nodes or
across all compute nodes. Centralizing north/south routing to a set of dedicated
network or gateway nodes is reminiscent of the legacy network node model. Enabling
all compute nodes as gateway chassis will narrow the failure domain and potential
bottlenecks at the cost of ensuring the computes can connect to the provider
networks.

The following section will describe how to configure your inventory to meet certain
deployment scenarios.

Deployment Scenarios
--------------------

OpenStack-Ansible supports the following common deployment scenarios:

- Collapsed Network/Gateway Nodes
- Collapsed Compute/Gateway Nodes
- Standalone Gateway Nodes

In an OpenStack-Ansible deployment, members of the ``network_hosts`` group are
considered members of the ``neutron_ovn_northd`` group and run OVN northd-related
services, while members of the ``compute_hosts`` group are considered members of
the ``neutron_ovn_controller`` group and run OVN controller-related services. It is
up to the deployer to dictate which nodes are considered "OVN Gateway Chassis" nodes
by using the ``network-gateway_hosts`` inventory group in ``openstack_user_config.yml``.

In ``openstack_user_config.yml``, specify the hosts or aliases that will run the
``ovn-controller`` service and act as an OVN Gateway Chassis, like so:

.. code-block:: yaml

  network-gateway_hosts:
    network-node1:
      ip: 172.25.1.21
    network-node2:
      ip: 172.25.1.22
    network-node3:
      ip: 172.25.1.23

Existing inventory aliases can also be used. In the following example, members of
the ``infrastructure_hosts`` group are also network hosts and will serve as
OVN Gateway Chassis nodes:

.. code-block:: yaml

  network-gateway_hosts: *infrastructure_hosts

In the following example, members of the ``compute_hosts`` group running the
``ovn-controller`` service will also serve as OVN Gateway Chassis nodes:

.. code-block:: yaml

  network-gateway_hosts: *compute_hosts

Lastly, specific hosts can also be targeted:

.. code-block:: yaml

  network-gateway_hosts:
    compute5:
      ip: 172.25.1.55
    compute10:
      ip: 172.25.1.60
    compute15:
      ip: 172.25.1.65
    compute20:
      ip: 172.25.1.70
    compute25:
      ip: 172.25.1.75

OpenStack-Ansible user variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To deploy OpenStack-Ansible using the ML2/OVN mechanism driver, set the
following user variables in the``/etc/openstack_deploy/user_variables.yml``
file:
 
.. code-block:: yaml

  neutron_plugin_type: ml2.ovn

  neutron_plugin_base:
    - ovn-router

  neutron_ml2_drivers_type: "vlan,local,geneve,flat"

The overrides are instructing Ansible to deploy the OVN mechanism driver and
associated OVN components. This is done by setting ``neutron_plugin_type``
to ``ml2.ovn``.

The ``neutron_plugin_base`` override enables Neutron to use OVN for
routing functions.

The ``neutron_ml2_drivers_type`` override provides support for all type
drivers supported by OVN.

Provider network overrides can be specified on a global or per-host basis,
and the following format can be used in ``user_variables.yml`` or per-host
in ``openstack_user_config.yml`` or host vars.

.. note::

  When ``network_interface_mappings`` are defined, the playbooks will attempt
  to connect the mapped interface to the respective OVS bridge. Omitting
  ``network_interface_mappings`` will require the operator to connect the
  interface to the bridge manually using the ``ovs-vsctl add-port`` command.

.. code-block:: yaml

  # When configuring Neutron to support geneve tenant networks and
  # vlan provider networks the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "geneve"
    network_geneve_ranges: "1:1000"
    network_vlan_ranges: "public"
    network_mappings: "public:br-publicnet"
    network_interface_mappings: "br-publicnet:bond1"

  # When configuring Neutron to support only vlan tenant networks and
  # vlan provider networks the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "vlan"
    network_vlan_ranges: "public:203:203,467:500"
    network_mappings: "public:br-publicnet"
    network_interface_mappings: "br-publicnet:bond1"

  # When configuring Neutron to support multiple vlan provider networks
  # the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "vlan"
    network_vlan_ranges: "public:203:203,467:500,private:101:200,301:400"
    network_mappings: "public:br-publicnet,private:br-privatenet"
    network_interface_mappings: "br-publicnet:bond1,br-privatenet:bond2"

(Optional) DVR or Distributed L3 routing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
DVR will be used for floating IPs if the ovn / enable_distributed_floating_ip
flag is configured to True in the neutron server configuration.

Create a group var file for neutron server
``/etc/openstack_deploy/group_vars/neutron_server.yml``. It has to include:

.. code-block:: yaml

  # DVR/Distributed L3 routing support
  neutron_neutron_conf_overrides:
    ovn:
      enable_distributed_floating_ip: True

Useful Open Virtual Network (OVN) Commands
------------------------------------------

The following commands can be used to provide useful information about the
state of Open vSwitch networking and configurations.

The ``ovs-vsctl list open_vswitch`` command provides information about the
``open_vswitch`` table in the local Open vSwitch database:

.. code-block:: console

  root@mnaio-controller1:~# ovs-vsctl list open_vswitch
  _uuid               : 7f96baf2-d75e-4a99-bb19-ca7138fc14c2
  bridges             : []
  cur_cfg             : 1
  datapath_types      : [netdev, system]
  datapaths           : {}
  db_version          : "8.3.0"
  dpdk_initialized    : false
  dpdk_version        : none
  external_ids        : {hostname=mnaio-controller1, rundir="/var/run/openvswitch", system-id="a67926f2-9543-419a-903d-23e2aa308368"}
  iface_types         : [bareudp, erspan, geneve, gre, gtpu, internal, ip6erspan, ip6gre, lisp, patch, stt, system, tap, vxlan]
  manager_options     : []
  next_cfg            : 1
  other_config        : {}
  ovs_version         : "2.17.2"
  ssl                 : []
  statistics          : {}
  system_type         : ubuntu
  system_version      : "20.04"

The ``ovn-sbctl show`` command provides information related to southbound
connections. If used outside the ovn_northd container, specify the
connection details:

.. code-block:: console

  root@mnaio-controller1:~# ovn-sbctl show
  Chassis "5335c34d-9233-47bd-92f1-fc7503270783"
      hostname: mnaio-compute1
      Encap geneve
          ip: "172.25.1.31"
          options: {csum="true"}
      Encap vxlan
          ip: "172.25.1.31"
          options: {csum="true"}
      Port_Binding "852530b5-1247-4ec2-9c39-8ae0752d2144"
  Chassis "ff66288c-5a7c-41fb-ba54-6c781f95a81e"
      hostname: mnaio-compute2
      Encap vxlan
          ip: "172.25.1.32"
          options: {csum="true"}
      Encap geneve
          ip: "172.25.1.32"
          options: {csum="true"}
  Chassis "cb6761f4-c14c-41f8-9654-16f3fc7cc7e6"
      hostname: mnaio-compute3
      Encap geneve
          ip: "172.25.1.33"
          options: {csum="true"}
      Encap vxlan
          ip: "172.25.1.33"
          options: {csum="true"}
      Port_Binding cr-lrp-022933b6-fb12-4f40-897f-745761f03186

The ``ovn-nbctl show`` command provides information about networks, ports,
and other objects known to OVN and demonstrates connectivity between the
northbound database and neutron-server.

.. code-block:: console

  root@mnaio-controller1:~# ovn-nbctl show
  switch 03dc4558-f83e-4531-b854-156292f1dbad (neutron-a6e65821-93e2-4521-9e31-37c35d52d953) (aka project-tenant-network)
      port 852530b5-1247-4ec2-9c39-8ae0752d2144
          addresses: ["fa:16:3e:d2:af:bf 10.3.3.49"]
      port 624de478-7e75-472f-b867-e6f514790a81
          addresses: ["fa:16:3e:bf:c0:c3 10.3.3.3", "unknown"]
      port 1cca8ef3-d3c9-4307-a779-13348db5e647
          addresses: ["fa:16:3e:4a:67:ed 10.3.3.4", "unknown"]
      port 05e20b32-2933-414a-ba31-eac683d09ac2
          addresses: ["fa:16:3e:bd:5d:e8 10.3.3.5", "unknown"]
      port 5a2e35cb-178b-443b-9f15-4c6ec4db4ac7
          type: router
          router-port: lrp-5a2e35cb-178b-443b-9f15-4c6ec4db4ac7
      port 2d52a2bf-ab37-4a18-87bd-8808a99c67d3
          type: localport
          addresses: ["fa:16:3e:30:b4:a0 10.3.3.2"]
  switch 3e03d5f1-4cfe-4c61-bd4c-8a661634d77b (neutron-b0b4017f-a9d1-4923-af35-944b88b7a393) (aka flat-external-provider-network)
      port 022933b6-fb12-4f40-897f-745761f03186
          type: router
          router-port: lrp-022933b6-fb12-4f40-897f-745761f03186
      port 347a7d8d-fd0f-48be-be02-d603258f0a08
          addresses: ["fa:16:3e:f4:6a:17 192.168.25.5", "unknown"]
      port 29c83838-329d-4839-bddb-818c7e2e9bc7
          addresses: ["fa:16:3e:a3:48:a8 192.168.25.3", "unknown"]
      port 173c9ceb-4dd3-4268-aaa3-c7b0f693a557
          type: localport
          addresses: ["fa:16:3e:0c:37:ed 192.168.25.2"]
      port 7a0175fd-ac09-4466-b3d0-26f696e3769c
          addresses: ["fa:16:3e:ad:19:c2 192.168.25.4", "unknown"]
      port provnet-525d3402-d582-49b4-b946-f28de8bbc615
          type: localnet
          addresses: ["unknown"]
  router 5ebb0cdb-2026-4454-a32e-eb5425ae7296 (neutron-b0d6ca32-fda3-4fdc-b648-82c8bee303dc) (aka project-router)
      port lrp-5a2e35cb-178b-443b-9f15-4c6ec4db4ac7
          mac: "fa:16:3e:3a:1c:bb"
          networks: ["10.3.3.1/24"]
      port lrp-022933b6-fb12-4f40-897f-745761f03186
          mac: "fa:16:3e:1f:cd:e9"
          networks: ["192.168.25.242/24"]
          gateway chassis: [cb6761f4-c14c-41f8-9654-16f3fc7cc7e6 ff66288c-5a7c-41fb-ba54-6c781f95a81e 5335c34d-9233-47bd-92f1-fc7503270783]
      nat 79d8486c-8b5e-4d6c-a56f-9f0df115f77f
          external ip: "192.168.25.242"
          logical ip: "10.3.3.0/24"
          type: "snat"
      nat d338ccdf-d3c4-404e-b2a5-938d0c212e0d
          external ip: "192.168.25.246"
          logical ip: "10.3.3.49"
          type: "dnat_and_snat"

Additional commands can be found in upstream OVN documentation and other
resources listed on this page.

Notes
-----

The ``ovn-controller`` service will check in as an agent and can be observed
using the ``openstack network agent list`` command:

.. code-block:: console

  +--------------------------------------+------------------------------+-------------------+-------------------+-------+-------+----------------------------+
  | ID                                   | Agent Type                   | Host              | Availability Zone | Alive | State | Binary                     |
  +--------------------------------------+------------------------------+-------------------+-------------------+-------+-------+----------------------------+
  | 5335c34d-9233-47bd-92f1-fc7503270783 | OVN Controller Gateway agent | mnaio-compute1    |                   | :-)   | UP    | ovn-controller             |
  | ff66288c-5a7c-41fb-ba54-6c781f95a81e | OVN Controller Gateway agent | mnaio-compute2    |                   | :-)   | UP    | ovn-controller             |
  | cb6761f4-c14c-41f8-9654-16f3fc7cc7e6 | OVN Controller Gateway agent | mnaio-compute3    |                   | :-)   | UP    | ovn-controller             |
  | 38206799-af64-589b-81b2-405f0cfcd198 | OVN Metadata agent           | mnaio-compute1    |                   | :-)   | UP    | neutron-ovn-metadata-agent |
  | 9e9b49c7-dd00-5f58-a3f5-22dd01f562c4 | OVN Metadata agent           | mnaio-compute2    |                   | :-)   | UP    | neutron-ovn-metadata-agent |
  | 72b1a6e2-4cca-570f-83a4-c05dcbbcc11f | OVN Metadata agent           | mnaio-compute3    |                   | :-)   | UP    | neutron-ovn-metadata-agent |
  +--------------------------------------+------------------------------+-------------------+-------------------+-------+-------+----------------------------+
