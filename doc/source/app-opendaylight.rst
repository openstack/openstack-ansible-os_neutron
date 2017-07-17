========================================
Scenario - OpenDaylight and Open vSwitch
========================================

Overview
~~~~~~~~

Deployers can choose to enhance neutron capabilities by means of the
OpenDaylight SDN Controller, which works together with Open vSwitch to provide
advanced networking capabilities. This document explains how to use them
in your environment.

Recommended reading
~~~~~~~~~~~~~~~~~~~

Since this is an extension of the basic Open vSwitch scenario, it is worth
reading that scenario to get some background. It is also recommended to be
familiar with OpenDaylight and networking-odl projects and their configuration.

 * `Scenario: Open vSwitch <app-openvswitch.html>`_
 * `OpenDaylight SDN Controller <docs.opendaylight.org/en/latest/>`_
 * `Networking-odl <https://github.com/openstack/networking-odl>`_

Prerequisites
~~~~~~~~~~~~~

The `OpenDaylight Ansible role <https://wiki.opendaylight.org/view/Deployment#Ansible_Role>`_
needs to be available in Ansible's role path.

OpenStack-Ansible user variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set the following user variables in your
``/etc/openstack_deploy/user_variables.yml``: *

.. code-block:: yaml

  ### Ensure the openvswitch kernel module is loaded
  openstack_host_specific_kernel_modules:
    - name: "openvswitch"
      pattern: "CONFIG_OPENVSWITCH"
      group: "network_hosts"

  ### Use OpenDaylight SDN Controller
  neutron_plugin_type: "ml2.opendaylight"
  neutron_opendaylight_conf_ini_overrides:
    ml2_odl:
      url: "http://<odl-ip>:<8080/controller/nb/v2/neutron"
      username: <username>
      password: <password>
      port_binding_controller: "pseudo-agentdb-binding"

Most of the content of this file is self-explanatory. The first block is used
to deploy Open vSwitch in all network hosts.

The second block is instructing Ansible to deploy OpenDaylight SDN Controller.
This is done by specifying ``neutron_plugin_type`` to ``ml2.opendaylight``.
After that, some configuration is needed to integrate OpenDaylight and Neutron,
using the ``ml2_odl`` section.

 * **url**: OpenDaylight's northbound url. This will be the same address as the
   first host where a neutron_server is running.
 * **username**: OpenDaylight northbound API username
 * **password**: OpenDaylight northbound API password for <username>
 * **port_binding_controller**: the method used for Neutron port binding,
   typically ``pseudo-agentdb-binding``.

Apart from these options, the deployer should keep in mind the installation
method for OpenDaylight Ansible role. It will be autoconfigured to ``deb_repo``
if the operating system detected belongs to the Debian family (e.g. Ubuntu),
and it will fall back to ``rpm_repo`` otherwise. Nevertheless, it can be forced
by the deployer.

Also, the set of features that will be enabled in the OpenDaylight SDN
controller defaults to ``odl-netvirt-openstack``, which is the minimum for an
OpenStack integration. The deployer can modify this value by providing a list
of feature names in the ``opendaylight_extra_features`` variable.

For more information, see OpenDaylight Ansible role documentation.

Security information
~~~~~~~~~~~~~~~~~~~~

Communications between the OpenDaylight SDN Controller and Open vSwitch are not
secured by default. For further information on securing this interface, see
these manuals:

TLS Support on OpenDaylight OpenFlow plugin:

    https://wiki.opendaylight.org/view/OpenDaylight_OpenFlow_Plugin:_TLS_Support

Secure Communication Between OpenFlow Switches and Controllers

    https://www.thinkmind.org/download.php?articleid=afin_2015_2_30_40047
