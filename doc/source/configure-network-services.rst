=======================================================
Configuring the Networking service (neutron) (optional)
=======================================================

The OpenStack Networking service (neutron) includes the following services:

Firewall as a Service (FWaaS)
  Provides a software-based firewall that filters traffic from the router.

Load Balancer as a Service (LBaaS)
  Provides load balancers that direct traffic to OpenStack instances or other
  servers outside the OpenStack deployment.

VPN as a Service (VPNaaS)
  Provides a method for extending a private network across a public network.

BGP Dynamic Routing service
  Provides a means for advertising self-service (private) network prefixes
  to physical network devices that support BGP.

SR-IOV Support
  Provides the ability to provision virtual or physical functions to guest
  instances using SR-IOV and PCI passthrough. (Requires compatible NICs)


Firewall service (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following procedure describes how to modify the
``/etc/openstack_deploy/user_variables.yml`` file to enable FWaaS.

Deploying FWaaS v1
------------------

#. Override the default list of neutron plugins to include
   ``firewall``:

   .. code-block:: yaml

      neutron_plugin_base:
        - firewall
        - ...

#. ``neutron_plugin_base`` is as follows:

   .. code-block:: yaml

      neutron_plugin_base:
         - router
         - firewall
         - neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
         - vpnaas
         - metering
         - qos

#. Execute the neutron install playbook in order to update the configuration:

   .. code-block:: shell-session

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-neutron-install.yml

#. Execute the horizon install playbook to show the FWaaS panels:

   .. code-block:: shell-session

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-horizon-install.yml

The FWaaS default configuration options may be changed through the
`conf override`_ mechanism using the ``neutron_neutron_conf_overrides``
dict.

Deploying FWaaS v2
------------------

FWaaS v2 is the next generation Neutron firewall service and will provide
a rich set of APIs for securing OpenStack networks. It is still under
active development.

Refer to the `FWaaS 2.0 API specification
<https://specs.openstack.org/openstack/neutron-specs/specs/newton/fwaas-api-2.0.html>`_
for more information on these FWaaS v2 features

Follow the steps below to deploy FWaaS v2:

.. note::
    FWaaS v1 and v2 cannot be deployed simultaneously.

#. Add the FWaaS v2 plugin to the ``neutron_plugin_base`` variable
   in ``/etc/openstack_deploy/user_variables.yml``:

   .. code-block:: yaml

      neutron_plugin_base:
        - router
        - metering
        - firewall_v2

   Ensure that ``neutron_plugin_base`` includes all of the plugins that you
   want to deploy with neutron in addition to the firewall_v2 plugin.

#. Run the neutron playbook to deploy the FWaaS v2 service plugin

   .. code-block:: console

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-neutron-install.yml


Load balancing service (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `neutron-lbaas`_ plugin for neutron provides a software load balancer
service and can direct traffic to multiple servers. The service runs as an
agent and it manages `HAProxy`_ configuration files and daemons.

The Newton release contains only the LBaaS v2 API. For more details about
transitioning from LBaaS v1 to v2, review the :ref:`lbaas-special-notes`
section below.

Deployers can make changes to the LBaaS default configuration options via the
``neutron_lbaas_agent_ini_overrides`` dictionary. Review the documentation on
the  `conf override`_ mechanism for more details.

.. _neutron-lbaas: https://wiki.openstack.org/wiki/Neutron/LBaaS
.. _HAProxy: http://www.haproxy.org/

Deploying LBaaS v2
------------------

#. Add the LBaaS v2 plugin to the ``neutron_plugin_base`` variable
   in ``/etc/openstack_deploy/user_variables.yml``:

   .. code-block:: yaml

      neutron_plugin_base:
        - router
        - metering
        - neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2

   Ensure that ``neutron_plugin_base`` includes all of the plugins that you
   want to deploy with neutron in addition to the LBaaS plugin.

   Adding the LBaaS v2 plugin to ``neutron_plugin_base`` automatically enables
   the Dashboard panels for LBaaS v2 when the ``os_horizon`` role is
   redeployed (see the following step).

#. Run the neutron playbook to deploy the LBaaS v2 agent and enable the
   Dashboard panels for LBaaSv2:

   .. code-block:: console

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-neutron-install.yml
       # openstack-ansible os-horizon-install.yml

.. _lbaas-special-notes:

Special notes about LBaaS
-------------------------

**LBaaS v1 was deprecated in the Mitaka release and is not available in the
Newton release.**

LBaaS v1 and v2 agents are unable to run at the same time. If you switch
LBaaS v1 to v2, the v2 agent is the only agent running. The LBaaS v1 agent
stops along with any load balancers provisioned under the v1 agent.

Load balancers are not migrated between LBaaS v1 and v2 automatically. Each
implementation has different code paths and database tables. You need
to manually delete load balancers, pools, and members before switching LBaaS
versions. Recreate these objects afterwards.

Virtual private network service (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following procedure describes how to modify the
``/etc/openstack_deploy/user_variables.yml`` file to enable VPNaaS.

#. Override the default list of neutron plugins to include
   ``vpnaas``:

   .. code-block:: yaml

      neutron_plugin_base:
        - router
        - metering

#. ``neutron_plugin_base`` is as follows:

   .. code-block:: yaml

      neutron_plugin_base:
         - router
         - metering
         - vpnaas

#. Override the default list of specific kernel modules
   in order to include the necessary modules to run ipsec:

   .. code-block:: yaml

      openstack_host_specific_kernel_modules:
         - { name: "ebtables", pattern: "CONFIG_BRIDGE_NF_EBTABLES=", group: "network_hosts" }
         - { name: "af_key", pattern: "CONFIG_NET_KEY=", group: "network_hosts" }
         - { name: "ah4", pattern: "CONFIG_INET_AH=", group: "network_hosts" }
         - { name: "ipcomp", pattern: "CONFIG_INET_IPCOMP=", group: "network_hosts" }

#. Execute the openstack hosts setup in order to load the kernel modules at
   boot and runtime in the network hosts

   .. code-block:: shell-session

      # openstack-ansible openstack-hosts-setup.yml --limit network_hosts\
      --tags "openstack_hosts-config"

#. Execute the neutron install playbook in order to update the configuration:

   .. code-block:: shell-session

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-neutron-install.yml

#. Execute the horizon install playbook to show the VPNaaS panels:

   .. code-block:: shell-session

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-horizon-install.yml

The VPNaaS default configuration options are changed through the
`conf override`_ mechanism using the ``neutron_neutron_conf_overrides``
dict.

.. _conf override: https://docs.openstack.org/openstack-ansible/latest/admin/openstack-operations.html

BGP Dynamic Routing service (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `BGP Dynamic Routing`_ plugin for neutron provides BGP speakers which can
advertise OpenStack project network prefixes to external network devices, such
as routers. This is especially useful when coupled with the `subnet pools`_
feature, which enables neutron to be configured in such a way as to allow users
to create self-service `segmented IPv6 subnets`_.

.. _BGP Dynamic Routing: https://docs.openstack.org/neutron/latest/admin/config-bgp-dynamic-routing.html
.. _subnet pools: https://docs.openstack.org/neutron/latest/admin/config-subnet-pools.html
.. _segmented IPv6 subnets: https://cloudbau.github.io/openstack/neutron/networking/2016/05/17/neutron-ipv6.html

The following procedure describes how to modify the
``/etc/openstack_deploy/user_variables.yml`` file to enable the BGP Dynamic
Routing plugin.

#. Add the BGP plugin to the ``neutron_plugin_base`` variable
   in ``/etc/openstack_deploy/user_variables.yml``:

   .. code-block:: yaml

      neutron_plugin_base:
        - ...
        - neutron_dynamic_routing.services.bgp.bgp_plugin.BgpPlugin

   Ensure that ``neutron_plugin_base`` includes all of the plugins that you
   want to deploy with neutron in addition to the BGP plugin.

#. Execute the neutron install playbook in order to update the configuration:

   .. code-block:: shell-session

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-neutron-install.yml


SR-IOV Support (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following procedure describes how to modify the OpenStack-Ansible
configuration to enable Neutron SR-IOV support.

.. _SR-IOV-Passthrough-For-Networking: https://wiki.openstack.org/wiki/SR-IOV-Passthrough-For-Networking


#. Define SR-IOV capable physical host interface for a provider network

  As part of every Openstack-Ansible installation, all provider networks
  known to Neutron need to be configured inside the
  ``/etc/openstack_deploy/openstack_user_config.yml`` file.
  For each supported network type (e.g. vlan), the attribute
  ``sriov_host_interfaces`` can be defined to map ML2 network names
  (``net_name`` attribute) to one or many physical interfaces.
  Additionally, the network will need to be assigned to the
  ``neutron_sriov_nic_agent`` container group.

Example configuration:

   .. code-block:: yaml

      provider_networks
        - network:
          container_bridge: "br-vlan"
          container_type: "veth"
          container_interface: "eth11"
          type: "vlan"
          range: "1000:2000"
          net_name: "physnet1"
          sriov_host_interfaces: "p1p1,p4p1"
          group_binds:
            - neutron_linuxbridge_agent
            - neutron_sriov_nic_agent

#. Configure Nova

   With SR-IOV, Nova uses PCI passthrough to allocate VFs and PFs to guest
   instances. Virtual Functions (VFs) represent a slice of a physical NIC,
   and are passed as virtual NICs to guest instances. Physical Functions
   (PFs), on the other hand, represent an entire physical interface and are
   passed through to a single guest.

   To use PCI passthrough in Nova, the ``PciPassthroughFilter`` filter
   needs to be added to the `conf override`_
   ``nova_scheduler_default_filters``.
   Finally, PCI devices available for passthrough need to be allow via
   the `conf override`_
   ``nova_pci_passthrough_whitelist``.

   Possible options which can be configured:

   .. code-block:: yaml

      # Single device configuration
      nova_pci_passthrough_whitelist: '{ "physical_network":"physnet1", "devname":"p1p1" }'

      # Multi device configuration
      nova_pci_passthrough_whitelist: '[{"physical_network":"physnet1", "devname":"p1p1"}, {"physical_network":"physnet1", "devname":"p4p1"}]'

      # Whitelisting by PCI Device Location
      # The example pattern for the bus location '0000:04:*.*' is very wide. Make sure that
      # no other, unintended devices, are whitelisted (see lspci -nn)
      nova_pci_passthrough_whitelist: '{"address":"0000:04:*.*", "physical_network":"physnet1"}'

      # Whitelisting by PCI Device Vendor
      # The example pattern limits matches to PCI cards with vendor id 8086 (Intel) and
      # product id 10ed (82599 Virtual Function)
      nova_pci_passthrough_whitelist: '{"vendor_id":"8086", "product_id":"10ed", "physical_network":"physnet1"}'

      # Additionally, devices can be matched by their type, VF or PF, using the dev_type parameter
      # and type-VF or type-PF options
      nova_pci_passthrough_whitelist: '{"vendor_id":"8086", "product_id":"10ed", "dev_type":"type-VF", physical_network":"physnet1"}'

   It is recommended to use whitelisting by either the Linux device name
   (devname attribute) or by the PCI vendor and product id combination
   (``vendor_id`` and ``product_id`` attributes)

#. Enable the SR-IOV ML2 plugin

   The `conf override`_ ``neutron_plugin_type`` variable defines the core
   ML2 plugin, and only one plugin can be defined at any given time.
   The `conf override`_ ``neutron_plugin_types`` variable can contain a list
   of additional ML2 plugins to load. Make sure that only compatible
   ML2 plugins are loaded at all times.
   The SR-IOV ML2 plugin is known to work with the linuxbridge (``ml2.lxb``)
   and openvswitch (``ml2.ovs``) ML2 plugins.
   ``ml2.lxb`` is the standard activated core ML2 plugin.

      .. code-block:: yaml

         neutron_plugin_types:
           - ml2.sriov


#. Execute the Neutron install playbook in order to update the configuration:

   .. code-block:: shell-session

       # cd /opt/openstack-ansible/playbooks
       # openstack-ansible os-neutron-install.yml
       # openstack-ansible os-nova-install.yml


#. Check Neutron SR-IOV agent state

   After the playbooks have finished configuring Neutron and Nova, the new
   Neutron Agent state can be verified with:

   .. code-block:: shell-session

       # neutron agent-list --agent_type 'NIC Switch agent'
       +--------------------------------------+------------------+-----------+-------+----------------+-------------------------+
       | id                                   | agent_type       | host      | alive | admin_state_up | binary                  |
       +--------------------------------------+------------------+-----------+-------+----------------+-------------------------+
       | 3012ff0e-de35-447b-aff6-fdb55b04c518 | NIC Switch agent | compute01 | :-)   | True           | neutron-sriov-nic-agent |
       | bb0c0385-394d-4e72-8bfe-26fd020df639 | NIC Switch agent | compute02 | :-)   | True           | neutron-sriov-nic-agent |
       +--------------------------------------+------------------+-----------+-------+----------------+-------------------------+


Deployers can make changes to the SR-IOV nic agent default configuration
options via the ``neutron_sriov_nic_agent_ini_overrides`` dict.
Review the documentation on the `conf override`_ mechanism for more details.

