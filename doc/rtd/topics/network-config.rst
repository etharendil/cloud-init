.. _network_config:

*********************
Network Configuration
*********************


Default Behavior
================

`Cloud-init`_ 's searches for network configuration in order of increasing
precedence; each item overriding the previous.

**Datasource**

For example, OpenStack may provide network config in the MetaData Service.

**System Config**

A ``network:`` entry in ``/etc/cloud/cloud.cfg.d/*`` configuration files.

**Kernel Command Line**

``ip=`` or ``network-config=<Base64 encoded YAML config string>``

User-data cannot change an instance's network configuration.  In the absence
of network configuration in any of the above sources , `Cloud-init`_ will
write out a network configuration that will issue a DHCP request on a "first"
network interface.

.. note::

   The network-config value is expected to be a Base64 encoded YAML string in
   :ref:`network_config_v1` or :ref:`network_config_v2` format. Optionally it
   can be compressed with ``gzip`` prior to Base64 encoding.


Disabling Network Configuration
===============================

Users may disable `Cloud-init`_ 's network configuration capability and rely
on other methods, such as embedded configuration or other customizations.

`Cloud-init`_ supports the following methods for disabling cloud-init.


**Kernel Command Line**

`Cloud-init`_ will check additionally check for the parameter
``network-config=disabled`` which will automatically disable any network
configuration.

Example disabling kernel command line entry: ::

  network-config=disabled


**cloud config**

In the combined cloud-init configuration dictionary, merged from
``/etc/cloud/cloud.cfg`` and ``/etc/cloud/cloud.cfg.d/*``::

  network:
    config: disabled

If `Cloud-init`_ 's networking config has not been disabled, and
no other network information is found, then it will proceed
to generate a fallback networking configuration.

Disabling Network Activation
----------------------------

Some datasources may not be initialized until after network has been brought
up. In this case, cloud-init will attempt to bring up the interfaces specified
by the datasource metadata using a network activator discovered by
`cloudinit.net.activators.select_activators`_.

This behavior can be disabled in the cloud-init configuration dictionary,
merged from ``/etc/cloud/cloud.cfg`` and ``/etc/cloud/cloud.cfg.d/*``::

  disable_network_activation: true

Fallback Network Configuration
==============================

`Cloud-init`_ will attempt to determine which of any attached network devices
is most likely to have a connection and then generate a network
configuration to issue a DHCP request on that interface.

`Cloud-init`_ runs during early boot and does not expect composed network
devices (such as Bridges) to be available.  `Cloud-init`_ does not consider
the following interface devices as likely 'first' network interfaces for
fallback configuration; they are filtered out from being selected.

- **loopback**: ``name=lo``
- **Virtual Ethernet**: ``name=veth*``
- **Software Bridges**: ``type=bridge``
- **Software VLANs**: ``type=vlan``


`Cloud-init`_ will prefer network interfaces that indicate they are connected
via the Linux ``carrier`` flag being set.  If no interfaces are marked
connected, then all unfiltered interfaces are potential connections.

Of the potential interfaces, `Cloud-init`_ will attempt to pick the "right"
interface given the information it has available.

Finally after selecting the "right" interface, a configuration is
generated and applied to the system.

.. note::

   PhotonOS disables fallback networking configuration by default leaving
   network unrendered when no other network config is provided.
   If fallback config is still desired on PhotonOS, it can be enabled by
   providing `disable_fallback_netcfg: false` in
   `/etc/cloud/cloud.cfg:sys_config` settings.

Network Configuration Sources
=============================

`Cloud-init`_ accepts a number of different network configuration formats in
support of different cloud substrates.  The Datasource for these clouds in
`Cloud-init`_ will detect and consume Datasource-specific network
configuration formats for use when writing an instance's network
configuration.

The following Datasources optionally provide network configuration:

- :ref:`datasource_config_drive`

  - `OpenStack Metadata Service Network`_
  - :ref:`network_config_eni`

- :ref:`datasource_digital_ocean`

  - `DigitalOcean JSON metadata`_

- :ref:`datasource_nocloud`

  - :ref:`network_config_v1`
  - :ref:`network_config_v2`
  - :ref:`network_config_eni`

- :ref:`datasource_opennebula`

  - :ref:`network_config_eni`

- :ref:`datasource_openstack`

  - :ref:`network_config_eni`
  - `OpenStack Metadata Service Network`_

- :ref:`datasource_smartos`

  - `SmartOS JSON Metadata`_

- :ref:`datasource_upcloud`

  - `UpCloud JSON metadata`_

- :ref:`datasource_vultr`

  - `Vultr JSON metadata`_

For more information on network configuration formats

.. toctree::
   :maxdepth: 1

   network-config-format-eni.rst
   network-config-format-v1.rst
   network-config-format-v2.rst


Network Configuration Outputs
=============================

`Cloud-init`_ converts various forms of user supplied or automatically
generated configuration into an internal network configuration state. From
this state `Cloud-init`_ delegates rendering of the configuration to Distro
supported formats.  The following ``renderers`` are supported in cloud-init:

- **NetworkManager**

`NetworkManager <https://networkmanager.dev>`_ is the standard Linux network
configuration tool suite. It supports a wide range of networking setups.
Configuration is typically stored in ``/etc/NetworkManager``.

It is the default for a number of Linux distributions, notably Fedora;
CentOS/RHEL; and derivatives.

- **ENI**

/etc/network/interfaces or ``ENI`` is supported by the ``ifupdown`` package
found in Alpine Linux, Debian and Ubuntu.

- **Netplan**

Introduced in Ubuntu 16.10 (Yakkety Yak), `netplan <https://netplan.io/>`_ has
been the default network configuration tool in Ubuntu since 17.10 (Artful
Aardvark).  netplan consumes :ref:`network_config_v2` input and renders
network configuration for supported backends such as ``systemd-networkd`` and
``NetworkManager``.

- **Sysconfig**

Sysconfig format is used by RHEL, CentOS, Fedora and other derivatives.


- **NetBSD, OpenBSD, FreeBSD**

Network renders supporting BSD releases which typically write configuration to
``/etc/rc.conf``. Unique to BSD renderers is that each renderer also calls
something akin to `FreeBSD.start_services`_ which will invoke applicable
network services to setup the network, making network activators unneeded
for BSD flavors at the moment.


Network Output Policy
=====================

The default policy for selecting a network ``renderer`` in order of preference
is as follows:

- ENI
- Sysconfig
- Netplan
- NetworkManager
- FreeBSD
- NetBSD
- OpenBSD
- Networkd

The default policy for selecting a network ``activator`` in order of preference
is as follows:
- ENI: using `ifup`, `ifdown` to manage device setup/teardown
- Netplan: using `netplan apply` to manage device setup/teardown
- NetworkManager: using `nmcli` to manage device setup/teardown
- Networkd: using `ip` to manage device setup/teardown


When applying the policy, `Cloud-init`_ checks if the current instance has the
correct binaries and paths to support the renderer.  The first renderer that
can be used is selected.  Users may override the network renderer policy by
supplying an updated configuration in cloud-config. ::

  system_info:
    network:
      renderers: ['netplan', 'network-manager', 'eni', 'sysconfig', 'freebsd', 'netbsd', 'openbsd']
      activators: ['eni', 'netplan', 'network-manager', 'networkd']


Network Configuration Tools
===========================

`Cloud-init`_ contains one tool used to test input/output conversion between
formats.  The ``tools/net-convert.py`` in the `Cloud-init`_ source repository
is helpful for examining expected output for a given input format.

CLI Interface :

.. code-block:: shell-session

  % tools/net-convert.py --help
  usage: net-convert.py [-h] --network-data PATH --kind
                        {eni,network_data.json,yaml} -d PATH [-m name,mac]
                        --output-kind {eni,netplan,sysconfig}

  optional arguments:
    -h, --help            show this help message and exit
    --network-data PATH, -p PATH
    --kind {eni,network_data.json,yaml}, -k {eni,network_data.json,yaml}
    -d PATH, --directory PATH
                          directory to place output in
    -m name,mac, --mac name,mac
                          interface name to mac mapping
    --output-kind {eni,netplan,sysconfig}, -ok {eni,netplan,sysconfig}


Example output converting V2 to sysconfig:

.. code-block:: shell-session

   $ tools/net-convert.py --network-data v2.yaml --kind yaml \
      --output-kind sysconfig -d target
   $ cat target/etc/sysconfig/network-scripts/ifcfg-eth*

Example output:

.. code-block::

   # Created by cloud-init on instance boot automatically, do not edit.
   #
   BOOTPROTO=static
   DEVICE=eth7
   IPADDR=192.168.1.5/255.255.255.0
   NM_CONTROLLED=no
   ONBOOT=yes
   TYPE=Ethernet
   USERCTL=no
   # Created by cloud-init on instance boot automatically, do not edit.
   #
   BOOTPROTO=dhcp
   DEVICE=eth9
   NM_CONTROLLED=no
   ONBOOT=yes
   TYPE=Ethernet
   USERCTL=no


.. _Cloud-init: https://launchpad.net/cloud-init
.. _DigitalOcean JSON metadata: https://developers.digitalocean.com/documentation/metadata/
.. _OpenStack Metadata Service Network: https://specs.openstack.org/openstack/nova-specs/specs/liberty/implemented/metadata-service-network-info.html
.. _SmartOS JSON Metadata: https://eng.joyent.com/mdata/datadict.html
.. _UpCloud JSON metadata: https://developers.upcloud.com/1.3/8-servers/#metadata-service
.. _Vultr JSON metadata: https://www.vultr.com/metadata/
.. _cloudinit.net.activators.select_activators: https://github.com/canonical/cloud-init/blob/main/cloudinit/net/activators.py#L279
.. _FreeBSD.start_services: https://github.com/canonical/cloud-init/blob/main/cloudinit/net/freebsd.py#L28

.. vi: textwidth=79
