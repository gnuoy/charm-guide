.. _new_sdn_charm:

New SDN Charm
=============

Before writing the charm the charm author needs to have a clear idea of what
applications the charm is going to need to relate to, what files and services
the charm is going to manage and possibly what files or services do other
charms manage that need updating.

In the example below we will assume that a new charm, VirtualTokenRing, is
needed to install a component on compute nodes and to inject some
configuration into nova.conf. 

Prerequists
===========

This will change once the OpenStack templates are on pypi

.. code:: bash

   git clone git@github.com:gnuoy/charm_templates_openstack.git
   cd charm_templates_openstack
   sudo ./setup.py install

Create Charm
============

Charm tools provides a utility for building an initial charm from a template.
During the charm generation charm tools asks a few questions about the charm.

.. code:: bash

    charm-create  -t openstack-neutron-plugin virtual-token-ring
    INFO: Generating charm for virtual-token-ring in ./virtual-token-ring
    INFO: No virtual-token-ring in apt cache; creating an empty charm instead.
    What is the earliest OpenStack release this charm is compatable with? liberty
    What packages should this charm install (space seperated list)?

.. _`Build Charm`:

Build Charm
===========

The charm now needs to be built to pull down all the interfaces and layers the
charm depends on and rolled into the built charm which can be deployed.

.. code:: bash

    cd virtual-token-ring
    charm build -s xenial -o build src

Deploy Charm
============

.. code:: bash

    cd build
    juju deploy cs:xenial/nova-compute
    juju deploy local:xenial/virtual-token-ring
    juju add-relation nova-compute virtual-token-ring
    
``juju status`` will now show both charms deployed. The ``nova-compute`` status
will show some missing relations but thats not an issue for this demonstration.


Updating nova.conf
==================

The charm will have installed any packages that were listed at charm creation
time but that is all it has done. The charm has access to its `neutron plugin
<https://github.com/openstack/charm-interface-neutron-plugin>`__ interface.
this allows the virtual-token-ring to send configuration to the nova-compute
charm for inclusion in the nova.conf.

Return to the **virtual-token-ring** directory and edit
**src/reactive/virtual_token_ring_handlers.py** adding the following:

.. code:: python

    @reactive.when('neutron-plugin.connected')
    def configure_neutron_plugin(neutron_plugin):
        neutron_plugin.configure_plugin(
            plugin='ovs',
            config={
                "nova-compute": {
                    "/etc/nova/nova.conf": {
                        "sections": {
                            'DEFAULT': [
                                ('random_option', 'true'),
                            ],
                        }
                    }
                }
            })

This tells the charm to send that configuation to the principle where the
**neutron-plugin.connected** event has been raised. Then repeat the `Build
Charm`_ steps/

Deploy Update
=============

The freshly built charm which contains the update now needs to be deployed to
the environment.

.. code:: bash

    juju upgrade-charm virtual-token-ring


Check Update
============

.. code:: bash

    juju run --unit nova-compute/0 "grep random_option /etc/nova/nova.conf"
    random_option = true


