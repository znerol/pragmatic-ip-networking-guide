Linux Router Example
********************

A setup as outlined in chapter :ref:`chapter-recommendations` can be implemented
using tools built into modern Linux distros by default. Most single-router
scenarios can be covered with ``systemd-networkd`` and ``nftables`` exclusively.

All configuration files are available in the github repo under
`examples/01-linux-router`_.

.. _`examples/01-linux-router`: https://github.com/znerol/pragmatic-ip-networking-guide/tree/main/examples/01-linux-router/


Network Layout
==============

Let's assume that the ISP assigned the IPv6 prefix ``2001:db8:1020:ff00::/56``.
Also this example uses ``10.20.0.0/16`` to allocate IPv4 subnets from.

.. table:: Interfaces, Subnet IDs (VLANs) and IP Ranges
   :widths: auto

   ==========  =====  ================  ========================  ==================
   Interface   Role   Subnet ID (VLAN)  IPv6 Prefix               IPv6 Address
   ==========  =====  ================  ========================  ==================
   ens1        wan    none              Assigned via SLAAC/DHCP6  Assigned via DHCP4
   ens2        trunk  none              none                      none
   vlan-dmz    dmz    158 (0x9e)        2001:db8:1020:ff9e::/64   none
   vlan-guest  guest  214 (0xd6)        2001:db8:1020:ffd6::/64   10.20.214.1/24
   vlan-staff  staff   84 (0x54)        2001:db8:1020:ff54::/64   10.20.54.1/24
   vpn         staff  244 (0xf4)        2001:db8:1020:fff4::/64   10.20.244.1/24
   ==========  =====  ================  ========================  ==================


Systemd network configuration
=============================

Network configuration is maintained in `systemd.network(5)`_ unit files under
``/etc/systemd/network``. The presented configuration makes extensive use of
per-network ``drop-in`` directories. This simplifies reuse of common
configuration snippets.

.. _systemd.network(5): https://manpages.debian.org/stable/systemd/systemd.network.5.en.html

.. code-block::

   $ tree network
   network
   ├── lo.network
   ├── lo.network.d
   │   ├── iface-type-loopback.conf
   │   └── inet-lo.conf
   ├── trunk.network
   ├── trunk.network.d
   │   ├── child-vlan-dmz.conf
   │   ├── child-vlan-guest.conf
   │   ├── child-vlan-staff.conf
   │   └── iface-type-trunk.conf
   ├── vlan-dmz.netdev
   ├── vlan-dmz.network
   ├── vlan-dmz.network.d
   │   ├── iface-type-router.conf
   │   └── inet-vlan-dmz.conf
   ├── vlan-guest.netdev
   ├── vlan-guest.network
   ├── vlan-guest.network.d
   │   ├── iface-service-dhcp4.conf
   │   ├── iface-service-router-adv.conf
   │   ├── iface-type-router.conf
   │   └── inet-vlan-guest.conf
   ├── vlan-staff.netdev
   ├── vlan-staff.network
   ├── vlan-staff.network.d
   │   ├── iface-service-dhcp4.conf
   │   ├── iface-service-router-adv.conf
   │   ├── iface-type-router.conf
   │   └── inet-vlan-staff.conf
   ├── vpn.netdev
   ├── vpn.netdev.d
   │   ├── peer1.example.com.conf
   │   └── peer2.example.com.conf
   ├── vpn.network
   ├── vpn.network.d
   │   └── inet-vpn.conf
   ├── wan.network
   └── wan.network.d
      └── inet-wan.conf

   8 directories, 31 files

Interface: ens1 / wan (autoconfigured via SLAAC/DHCP)
-----------------------------------------------------

The wan network consists of the ``wan.network`` unit file (containing the
``Match`` section) and the ``wan.network.d/inet-wan.conf`` drop-in (specifying
how ip4/ip6 addressing is performed).

.. literalinclude:: ../examples/01-linux-router/network/wan.network
   :caption: wan.network
   :language: ini

.. literalinclude:: ../examples/01-linux-router/network/wan.network.d/inet-wan.conf
   :caption: wan.network.d/inet-wan.conf
   :language: ini

Interface: ens2 / trunk
-----------------------

The trunk network consists of the ``trunk.network`` unit file (containing the
``Match`` section) and several drop-ins.

.. literalinclude:: ../examples/01-linux-router/network/trunk.network
   :caption: trunk.network
   :language: ini

``trunk.network.d/iface-type-trunk.conf`` simply disables IPv6 link-local
addresses.

.. literalinclude:: ../examples/01-linux-router/network/trunk.network.d/iface-type-trunk.conf
   :caption: trunk.network.d/iface-type-trunk.conf
   :language: ini

For each VLAN, a ``child-vlan-XXX.conf`` ensures that the specified
VLAN is added to the trunk interface.

.. literalinclude:: ../examples/01-linux-router/network/trunk.network.d/child-vlan-dmz.conf
   :caption: trunk.network.d/child-vlan-dmz.conf
   :language: ini

.. literalinclude:: ../examples/01-linux-router/network/trunk.network.d/child-vlan-guest.conf
   :caption: trunk.network.d/child-vlan-guest.conf
   :language: ini

.. literalinclude:: ../examples/01-linux-router/network/trunk.network.d/child-vlan-staff.conf
   :caption: trunk.network.d/child-vlan-staff.conf
   :language: ini

VLAN: vlan-dmz (IPv6 only, static addressing)
---------------------------------------------

VLAN devices are created using `systemd.netdev(5)`_ units.

.. _systemd.netdev(5): https://manpages.debian.org/stable/systemd/systemd.netdev.5.en.html

.. literalinclude:: ../examples/01-linux-router/network/vlan-dmz.netdev
   :caption: vlan-dmz.netdev
   :language: ini

The specified device name is then used in the ``Match`` section of the
corresponding ``network`` unit.

.. literalinclude:: ../examples/01-linux-router/network/vlan-dmz.network
   :caption: vlan-dmz.network
   :language: ini

As pointed out in chapter :ref:`chapter-recommendations` it can be beneficial to
use a fixed ``fe80::1`` link-local address on router interfaces. The drop-in
``iface-type-router.conf`` provides the necessary settings. Additionally it
disables acceptance of router advertisements on this interface and enables
forwarding.

.. literalinclude:: ../examples/01-linux-router/network/vlan-dmz.network.d/iface-type-router.conf
   :caption: vlan-dmz.network.d/iface-type-router.conf
   :language: ini

The second drop-in ``inet-vlan-dmz.conf`` adds a route to the DMZ subnet
(``2001:db8:1020:ff9e::/64``) to this interface. Note that link-local addresses
are used for routing in IPv6. Hence, it is not necessary to actually assign a
globally routed address to router interfaces.

Since this network segment is IPv6 only, there is no need to add IPv4 addresses
/ routes to this interface.

.. literalinclude:: ../examples/01-linux-router/network/vlan-dmz.network.d/inet-vlan-dmz.conf
   :caption: vlan-dmz.network.d/inet-vlan-dmz.conf
   :language: ini

VLAN: vlan-staff (Dual-stack, SLAAC und DHCP4)
-------------------------------------------------------------------------

Base files like ``vlan-staff.netdev`` and ``vlan-staff.network`` work analogous
to the example above. The ``iface-type-router.conf`` drop-in can be reused
without modification. Since this interface needs to provide IPv4 connectivity,
an appropriate address needs to be supplied via ``inet-vlan-staff.conf``
drop-in.

.. literalinclude:: ../examples/01-linux-router/network/vlan-staff.network.d/inet-vlan-staff.conf
   :caption: vlan-staff.network.d/inet-vlan-staff.conf
   :language: ini

Another set of drop-ins is used to configure router advertisements:

.. literalinclude:: ../examples/01-linux-router/network/vlan-staff.network.d/iface-service-router-adv.conf
   :caption: vlan-staff.network.d/iface-service-router-adv.conf
   :language: ini

And DHCP4 server:

.. literalinclude:: ../examples/01-linux-router/network/vlan-staff.network.d/iface-service-dhcp4.conf
   :caption: vlan-staff.network.d/iface-service-dhcp4.conf
   :language: ini

Note that neither ``iface-service-router-adv.conf`` nor
``iface-service-dhcp4.conf`` contain any interface specific configuration.
Hence, they can be reused again for the ``vlan-guest`` interface.

Wireguard: vpn
--------------

Wireguard specific settings are mantained in ``netdev`` units. This includes key
material, the listen port and peer definitions.

.. literalinclude:: ../examples/01-linux-router/network/vpn.netdev
   :caption: vpn.netdev
   :language: ini

In order to simplify management of peers, configuration for each peer should be
maintained in a separate ``drop-in`` file.

.. literalinclude:: ../examples/01-linux-router/network/vpn.netdev.d/peer1.example.com.conf
   :caption: vpn.netdev.d/peer1.example.com.conf
   :language: ini

.. literalinclude:: ../examples/01-linux-router/network/vpn.netdev.d/peer2.example.com.conf
   :caption: vpn.netdev.d/peer2.example.com.conf
   :language: ini

.. caution:: **netdev configuration cannot be reloaded**

   Most configuration can be applied at runtime using ``networkctl reload``.
   However, configuration for ``netdev`` units is only applied upon creation of
   virtual devices. As a result, in order to apply wireguard configuration after
   a peer was added or removed, it is regrettably necessary to completely remove
   the wireguard interface before ``networkd`` picks up the new config. I.e.:

   .. code-block:: sh

      ip link del vpn
      networkctl releoad

   See `systemd/systemd#9627`_ for more details.

Network configuration via network units / drop-ins for wireguard interface
follows the same pattern as all examples presented here. The ``Match`` section
matches the name from the ``netdev`` unit. The ``inet-vpn.conf`` drop-in adds
IPv6 and IPv4 addresses acting as the gateway for connected clients.

.. literalinclude:: ../examples/01-linux-router/network/vpn.network
   :caption: vpn.network
   :language: ini

.. literalinclude:: ../examples/01-linux-router/network/vpn.network.d/inet-vpn.conf
   :caption: vpn.network.d/inet-vpn.conf
   :language: ini

.. _`systemd/systemd#9627`: https://github.com/systemd/systemd/issues/9627

Loopback: lo
------------

Note that no static globally routable IPv6 address was assigned to any interface
(except for the vpn gateway). In order to access services on the router
(including SSH), a static IPv6 address needs to be present at some interface.

Networking folks developed the habit to assign a routable IP on the loopback
interface. This is especially useful on devices with many interfaces in the
context of dynamic routing. The loopback interface never goes down, and thus an
IP assigned to ``lo`` will be reachable as long as there is at least one route.

Analogous to earlier examples ``lo.network`` simply matches the loopback device.
The ``iface-type-loopback.conf`` drop-in is responsible for device type specific
config. Notable ``KeepConfiguration = static`` preserves existing IP addresses
and routes (i.e., ``127.0.0.1/8`` and ``::1/128`` configured during system
bootup).

.. literalinclude:: ../examples/01-linux-router/network/lo.network.d/iface-type-loopback.conf
   :caption: lo.network.d/iface-type-loopback.conf
   :language: ini

The ``inet-lo.conf`` drop-in just assigns the routers IP:

.. literalinclude:: ../examples/01-linux-router/network/lo.network.d/inet-lo.conf
   :caption: lo.network.d/inet-lo.conf
   :language: ini

This IP should be recorded in DNS. It can be used to ``ping`` and ``ssh`` from
wherever there is IPv6 connectivity - as long as filter rules allow it.

Nftables Ruleset
================

Documentation on nftables regrettably isn't that comprehensive yet. The
`nft(8)`_ manpage provides up-to-date reference material. Some usage examples
and recipes are available from the `nftables wiki`_ and also from various Linux
distro wiki pages (quality of content varies). It is essential to understand
`packet flow through netfilter hooks`_ and to keep in mind the following rule
when reasoning about rulesets:

.. hint:: **Evaluation of Rules**

   In order to be delivered, a packet must be ``accept``-ed by every base chain
   in all traversed hooks.

   A packet is discarded immediately as soon as it is ``drop``-ed or
   ``reject``-ed. None of the rules in later chains and hooks will have the
   opportunity to further handle it.

.. _`nft(8)`: https://manpages.debian.org/stable/nftables/nft.8.en.html
.. _`nftables wiki`: https://wiki.nftables.org/wiki-nftables/index.php/Main_Page
.. _`packet flow through netfilter hooks`: https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks
