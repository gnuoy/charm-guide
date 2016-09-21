.. _new_api_charm:

New API Charm
=============

Before writing the charm the charm author needs to have a clear idea of what
applications the charm is going to need to relate to, what files and services
the charm is going to manage and possibly what files or services do other
charms manage that need updating.

In the example below we will walk through coding a basic Congress charm

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

.. code::

    charm-create  -t openstack-api-congress
    What port does the primary service listen on ? 1789
    What is the name of the api service? congress-api
    What type of service is this (used for keystone registration)? congress
    What is the earliest OpenStack release this charm is compatable with?  mitaka
    Where command is used to sync the database? congress-db-manage --config-file /etc/congress/congress.conf upgrade head
    Where is the primary configuration file for this service?  /etc/congress/congress.conf
    What packages should this charm install (space seperated list)?  congress-server congress-common python-antlr3 python-pymysql
    What is the name of the init script which controls the primary service?  congress-server

.. _`Build Charm`:

Build Charm
===========

The charm now needs to be built to pull down all the interfaces and layers the
charm depends on and rolled into the built charm which can be deployed.

.. code:: bash

    cd congress
    charm build -s xenial -o build src

Deploy Charm
============

Asumming that an OpenStack cloud is already deployed, add the new Congress
charm.

.. code:: bash

    cd build
    juju deploy local:xenial/congress
    juju add-relation congress keystone
    juju add-relation congress rabbitmq-server
    juju add-relation congress mysql
    
``juju status`` will show the deployment as it proceeds.

Configuration Files
===================

The charm code searches through the templates directories looking for a directory
corresponding to the Openstack release being installed or earlier. Since Mitaka
is the earliest release the charm is supporting a directory called mitaka will
house the templates and files.

.. code:: bash

    cd congress
    ( cd /tmp; dget -u http://archive.ubuntu.com/ubuntu/pool/universe/c/congress/congress_3.0.0+dfsg1-1.dsc;)
    mkdir -p src/templates/mitaka
    cp /tmp/congress*/etc/{api-paste.ini,policy.json} src/templates/mitaka

A template for congress.conf is needed which will have have connection
information for MySQL and Keystone as well as user controllable
config options. Create **src/templates/mitaka/congress.conf** with the
following contents:

.. code:: bash

    [DEFAULT]
    auth_strategy = keystone
    drivers = congress.datasources.neutronv2_driver.NeutronV2Driver,congress.datasources.glancev2_driver.GlanceV2Driver,congress.datasources.nova_driver.NovaDriver,congress.datasources.keystone_driver.KeystoneDriver,congress.datasources.ceilometer_driver.CeilometerDriver,congress.datasources.cinder_driver.CinderDriver,congress.datasources.swift_driver.SwiftDriver,congress.datasources.plexxi_driver.PlexxiDriver,congress.datasources.vCenter_driver.VCenterDriver,congress.datasources.murano_driver.MuranoDriver,congress.datasources.ironic_driver.IronicDriver

    [database]
    connection = {{ shared_db.uri }}

    {% include "section-keystone-authtoken-mitaka" %}

Deploy Update
=============

The freshly built charm which contains the update now needs to be deployed to
the environment.

.. code:: bash

    juju upgrade-charm congress

Add Relations
=============

.. code:: bash

    juju add-relation congress keystone
    juju add-relation congress mysql


Check Update
============

.. code:: bash

    juju run --unit nova-compute/0 "grep random_option /etc/nova/nova.conf"
    random_option = true


