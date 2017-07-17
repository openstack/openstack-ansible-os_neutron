==================================
Neutron role for OpenStack-Ansible
==================================

.. toctree::
   :maxdepth: 2

   configure-network-services.rst
   app-openvswitch.rst
   app-nuage.rst
   app-calico.rst
   app-opendaylight.rst

:tags: openstack, neutron, cloud, ansible
:category: \*nix

This role installs the following Systemd services:

  * neutron-server
  * neutron-agents

To clone or view the source code for this repository, visit the role repository
for `os_neutron <https://github.com/openstack/openstack-ansible-os_neutron>`_.

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.

Required variables
~~~~~~~~~~~~~~~~~~

None.

Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml

Tags
~~~~

This role supports two tags: ``neutron-install`` and
``neutron-config``. The ``neutron-install`` tag can be used to install
and upgrade. The ``neutron-config`` tag can be used to maintain the
configuration of the service.
