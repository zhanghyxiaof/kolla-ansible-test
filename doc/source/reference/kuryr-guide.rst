==============
Kuryr in Kolla
==============

"Kuryr is a Docker network plugin that uses Neutron to provide networking
services to Docker containers. It provides containerized images for the common
Neutron plugins. Kuryr requires at least Keystone and neutron. Kolla makes
kuryr deployment faster and accessible.

Requirements
~~~~~~~~~~~~

* A minimum of 3 hosts for a vanilla deploy

Preparation and Deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~

To allow Docker daemon connect to the etcd, add the following in the
``docker.service`` file.

.. code-block:: none

   ExecStart= -H tcp://172.16.1.13:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://172.16.1.13:2379 --cluster-advertise=172.16.1.13:2375

.. end

The IP address is host running the etcd service. ```2375``` is port that
allows Docker daemon to be accessed remotely. ```2379``` is the etcd listening
port.

By default etcd and kuryr are disabled in the ``group_vars/all.yml``.
In order to enable them, you need to edit the file globals.yml and set the
following variables

.. code-block:: yaml

   enable_etcd: "yes"
   enable_kuryr: "yes"

.. end

Deploy the OpenStack cloud and kuryr network plugin

.. code-block:: console

   kolla-ansible deploy

.. end

Create a Virtual Network
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   docker network create -d kuryr --ipam-driver=kuryr --subnet=10.1.0.0/24 --gateway=10.1.0.1 docker-net1

.. end

To list the created network:

.. code-block:: console

   docker network ls

.. end

The created network is also available from OpenStack CLI:

.. code-block:: console

   openstack network list

.. end

For more information about how kuryr works, see
`kuryr (OpenStack Containers Networking)
<https://docs.openstack.org/kuryr/latest/>`__.
