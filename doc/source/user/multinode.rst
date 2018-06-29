.. _multinode:

=============================
Multinode Deployment of Kolla
=============================

.. _deploy_a_registry:

Deploy a registry
=================

A Docker registry is a locally hosted registry that replaces the need to pull
from the Docker Hub to get images. Kolla can function with or without a local
registry, however for a multinode deployment some type of registry is
mandatory.  Only one registry must be deployed, although HA features exist for
registry services.

The Docker registry prior to version 2.3 has extremely bad performance because
all container data is pushed for every image rather than taking advantage of
Docker layering to optimize push operations. For more information reference
`pokey registry <https://github.com/docker/docker/issues/14018>`__.

Edit the ``/etc/kolla/globals.yml`` and add the following where 192.168.1.100
is the IP address of the machine and 5000 is the port where the registry is
currently running:

.. code-block:: none

   docker_registry = 192.168.1.100:5000

.. end

The Kolla community recommends using registry 2.3 or later. To deploy registry
with version 2.3 or later, do the following:

.. code-block:: console

   cd kolla
   tools/start-registry

.. end

The Docker registry can be configured as a pull through cache to proxy the
official Kolla images hosted in Docker Hub. In order to configure the local
registry as a pull through cache, in the host machine set the environment
variable ``REGISTRY_PROXY_REMOTEURL`` to the URL for the repository on
Docker Hub.

.. code-block:: console

   export REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io

.. end

.. note::

   Pushing to a registry configured as a pull-through cache is unsupported.
   For more information, Reference the `Docker Documentation
   <https://docs.docker.com/registry/configuration/>`__.

.. _configure_docker_all_nodes:

Configure Docker on all nodes
=============================

.. note::

   As the subtitle for this section implies, these steps should be
   applied to all nodes, not just the deployment node.

After starting the registry, it is necessary to instruct Docker that
it will be communicating with an insecure registry.
For example, To enable insecure registry communication on CentOS,
modify the ``/etc/sysconfig/docker`` file to contain the following where
``192.168.1.100`` is the IP address of the machine where the registry
is currently running:

.. path /etc/sysconfig/docker
.. code-block:: ini

   INSECURE_REGISTRY="--insecure-registry 192.168.1.100:5000"

.. end

For Ubuntu, check whether its using upstart or systemd.

.. code-block:: console

   # stat /proc/1/exe
   File: '/proc/1/exe' -> '/lib/systemd/systemd'

Edit ``/etc/default/docker`` and add the following configuration:

.. path /etc/default/docker
.. code-block:: ini

   DOCKER_OPTS="--insecure-registry 192.168.1.100:5000"

.. end

If Ubuntu is using systemd, additional settings needs to be configured.
Copy Docker's systemd unit file to ``/etc/systemd/system/`` directory:

.. code-block:: console

   cp /lib/systemd/system/docker.service /etc/systemd/system/docker.service

.. end

Next, modify ``/etc/systemd/system/docker.service``, add ``environmentFile``
variable and add ``$DOCKER_OPTS`` to the end of ExecStart in ``[Service]``
section.

For CentOS:

.. path /etc/systemd/system/docker.service
.. code-block:: ini

    [Service]
    MountFlags=shared
    EnvironmentFile=/etc/sysconfig/docker
    ExecStart=
    ExecStart=/usr/bin/docker daemon $INSECURE_REGISTRY

.. end

For Ubuntu:

.. path /etc/systemd/system/docker.service
.. code-block:: ini

   [Service]
   MountFlags=shared
   EnvironmentFile=-/etc/default/docker
   ExecStart=
   ExecStart=/usr/bin/docker daemon -H fd:// $DOCKER_OPTS

.. end

.. note::

   If your docker version is >=1.13.0, the ``docker daemon`` should be replaced
   with ``dockerd``.


Restart Docker by executing the following commands:

For CentOS or Ubuntu with systemd:

.. code-block:: console

   systemctl daemon-reload
   systemctl restart docker

.. end

For Ubuntu with upstart or sysvinit:

.. code-block:: console

   service docker restart

.. end

.. _edit-inventory:

Edit the Inventory File
=======================

The ansible inventory file contains all the information needed to determine
what services will land on which hosts. Edit the inventory file in the
Kolla-Ansible directory ``ansible/inventory/multinode``. If Kolla-Ansible
was installed with pip, it can be found in ``/usr/share/kolla-ansible``.

Add the IP addresses or hostnames to a group and the services associated with
that group will land on that host. IP addresses or hostnames must be added to
the groups control, network, compute, monitoring and storage. Also, define
additional behavioral inventory parameters such as ``ansible_ssh_user``,
``ansible_become`` and ``ansible_private_key_file/ansible_ssh_pass`` which
controls how ansible interacts with remote hosts.

.. note::

   Ansible uses SSH to connect the deployment host and target hosts. For more
   information about SSH authentication please reference
   `Ansible documentation <http://docs.ansible.com/ansible/intro_inventory.html>`__.

.. code-block:: none

   # These initial groups are the only groups required to be modified. The
   # additional groups are for more control of the environment.
   [control]
   # These hostname must be resolvable from your deployment host
   control01      ansible_ssh_user=<ssh-username> ansible_become=True ansible_private_key_file=<path/to/private-key-file>
   192.168.122.24 ansible_ssh_user=<ssh-username> ansible_become=True ansible_private_key_file=<path/to/private-key-file>

.. end

.. note::

   Additional inventory parameters might be required according to your
   environment setup. Reference `Ansible Documentation
   <http://docs.ansible.com/ansible/intro_inventory.html>`__ for more
   information.


For more advanced roles, the operator can edit which services will be
associated in with each group. Keep in mind that some services have to be
grouped together and changing these around can break your deployment:

.. code-block:: none

   [kibana:children]
   control

   [elasticsearch:children]
   control

   [haproxy:children]
   network

.. end

Deploying Kolla
===============

.. note::

    If there are multiple keepalived clusters running within the same layer 2
    network, edit the file ``/etc/kolla/globals.yml`` and specify a
    ``keepalived_virtual_router_id``. The ``keepalived_virtual_router_id`` should
    be unique and belong to the range 0 to 255.

.. note::

   If glance is configured to use ``file`` as backend, only one ``glance_api``
   container will be started. ``File`` is enabled by default when no other
   backend is specified in globals.yml

First, check that the deployment targets are in a state where Kolla may deploy
to them:

.. code-block:: console

   kolla-ansible prechecks -i <path/to/multinode/inventory/file>

.. end

.. note::

   RabbitMQ doesn't work with IP addresses, hence the IP address of
   ``api_interface`` should be resolvable by hostnames to make sure that all
   RabbitMQ Cluster hosts can resolve each others hostnames beforehand.

Run the deployment:

.. code-block:: console

   kolla-ansible deploy -i <path/to/multinode/inventory/file>

.. end
