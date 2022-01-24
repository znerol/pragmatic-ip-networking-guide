.. _chapter-recommendations:

Recommendations
***************

Segments
========

Unless there is only one client, networks need to be segmented into
multiple subnets on different VLANs. Different classes of nodes require
different services from the network. Separating them makes it easier to meet
those requirements.

Use separate VLANs for (non exhaustive list):

- Printers
- IoT devices
- VoIP phones
- Servers, VMs, Containers
- Wi-Fi controllers and access points
- Laptops and Workstations
- Temporary devices / Guests
- Experiments

.. hint:: **Segments and Autoconfiguration**

   Do not mix statically addressed and autoconfigured nodes in the same
   subnet.

   Laptops and Workstations typically are configured using router advertisements
   and/or DHCP while static addresses should be assigned to servers. Keeping
   dynamic and static clients in separate subnets prevents potential address
   collision without additional configuration overhead. It also allows for
   servers to deactivate DAD (duplicate address detection).

.. hint:: **Dual stacking only when needed**

   Printers placed in a separate subnet are easier to isolate from the internet
   (if necessary). In some situations, IPv4 might not be needed at all for a
   printer subnet, hence such a network segment can be operated using IPv6
   exclusively. Same goes for other internal-only services (e.g., media server).

Networks and Routes
===================

Subnets
-------

An IPv6 prefix with a size of ``/60`` can be divided into 16 ``/64`` subnets, a
prefix with the size of ``/56`` has space for 256 subnets and a ``/48`` prefix
holds 65536 subnets. This guide is going to use ``2001:db8:1020:ff00::/56`` as
the block used to allocate subnets from. Note, the IPv6 address range is
assigned by the upstream provider. It cannot be chosen freely.

If IPv4 is still a thing when you read this guide, choose an IPv4 block from
`RFC 1918`_ big enough to be split into several ``/24`` networks. It is best to
avoid blocks which include ranges used broadly in factory defaults like
``192.168.0.0/24``, ``192.168.1.0/24`` and ``10.0.0.0/24``. Instead opt for
lesser used IPs. This guide is going to use ``10.20.0.0/16`` as the block used
to allocate subnets from.

.. hint:: **VLAN id == subnet id**

   When creating new network segments, choose the VLAN id to be identical to the
   subnet id. E.g. for a new network segment with VLAN id 8 allocate
   ``2001:db8:1020:ff08::/64`` for the IPv6 range and ``10.20.8.0/24`` for IPv4.
   For a new network segment with VLAN id 33 (hex: 0x21) allocate
   ``2001:db8:1020:ff21::/64`` for the IPv6 range and ``10.20.33.0/24`` for
   IPv4. Note: If the IPv6 prefix is a ``/48``, then it is also possible to use
   the VLAN id in decimal notation for the IPv6 subnet.

.. _RFC 1918: https://datatracker.ietf.org/doc/html/rfc1918

Default Gateway
---------------

IPv6 routers advertise their link-local address and not a globally routable one
(see `RFC 4861 section 6.1.2`_). Hence, the default gateway on connected nodes
is supposed to point to a link local address (i.e., an address within
``fe80::/64``). Usually, IPv6 link-local addresses are derived automatically
from the MAC address of the network interface. However, using generated
addresses is not very practical in server subnets where hosts are configured
statically. Therefore it is advisable to manually configure the link-local
address for every subnet on routers.

.. hint:: **Use fe80::1/64**

   Fortunately link-local addresses are link-local. Therefore, it is perfectly
   valid to use the same address (i.e., ``fe80::1/64``) on every interface of a
   router (see: `Blog post: fe80::1 is a Perfectly Valid IPv6 Default Gateway
   Address`_)

.. _RFC 4861 section 6.1.2: https://datatracker.ietf.org/doc/html/rfc4861#section-6.1.2
.. _`Blog post: fe80::1 is a Perfectly Valid IPv6 Default Gateway Address`: https://blogs.infoblox.com/ipv6-coe/fe80-1-is-a-perfectly-valid-ipv6-default-gateway-address/

Names
=====

Most people cannot be bothered to remember IPv4 addresses. IPv6 does not make
things any easier.

DNS Hosting
-----------

DNS zones need to be hosted somewhere. SOHO routers typically provide point and
click interfaces where DNS entries can be added to some internal DNS server
which typically doubles as recursive DNS resolver. In order for this to work,
the address of that internal DNS server needs to be configured (maunally or via
autoconfiguration) on all nodes who wish to resolve those entries.

This configuration is known as split-horizon DNS. And it falls short if people
start to use validating resolvers (DNSSEC) and alternative DNS resolvers,
sometimes over encrypted protocols (DoT, DoH).

.. hint:: **Place RRs in public DNS**

   Avoid deploying private DNS zones and split-horizon DNS. Instead place all
   resource records into public DNS.

Node Names
----------

Every node which provides any type of service should have a name. This
includes routers, servers, VMs, containers, printers, WiFi access points etc. A
popular choice is to just add ``AAAA`` records to the domain name of the
family website or the primary domain of a business.

While convenient in the beginning, this can pose problems down the road. The
website might be managed by contractors while the network stays inhouse or
vice-versa. Node records might be managed by an orchestrator while access to
the DNS zone of the main website needs to be restricted due to policy reasons.
Maintaining separate DNS zones for different purposes also simplifies gradual
rollouts, e.g. of DNSSEC.

.. hint:: **Use a dedicated domain for nodes**

   Thus it is recommended to register and maintain a dedicated domain and only
   add ``AAAA`` records for network nodes there. Additional records like
   ``SSHFP`` could be added as well, this will simplify node administration
   greatly.

Nodes need to be replaced over time. In order to simplify this process, old
names should not be reused for new nodes. Instead each node keeps its name
over its whole lifespan in a network. Holding on to this practice simplifies
the development of a network since old and new equipment can be operated in
parallel for some time.

Service Names
-------------

Every service should have a name. This includes webapps, file sharing,
directory services, etc. A popular choice is to just use the node name where
the service happens to be hosted.

Services need to keep their name, otherwise people are forced to update
bookmarks and printer queues. It follows that reusing node names for services
will pose problems in the long run when nodes need to be replaced.

In addition, services might be composed from several applications running on
different nodes, VMs or containers. The service name is then simply pointing
towards the node hosting the frontend server for TLS termination, reverse
proxying and/or load balancing.

.. hint:: **Use a dedicated domain for services**

   Thus it is recommended to register and maintain a dedicated domain for
   internal services. Either add ``AAAA`` records containing IPs of the nodes
   hosting a service or add ``CNAME`` pointing towards the node names.

.. caution:: **Service enumeration via CT logs**

   TLS certificates issued by trusted certificate authorities are recorded in
   public certificate transparency logs (e.g. `crt.sh`_). Organisations which
   are reusing subdomains of their main website or brand name for internal
   systems secured by TLS certificates might unknowingly expose this information
   to the public.

   Using a dedicated domain name for internal services unrelated to the main
   website, name or brand of an organisation and deploying wildcard TLS
   certificate can reduce the risk of service enumeration via certificate
   transparency logs.

.. _crt.sh: https://crt.sh/

Addressing
==========

Nodes providing services to connected clients need a fixed IP address. IPv6
addresses assigned automatically via SLAAC do not change over time. Thus, such
IPs are quite suitable as a stable identifier for a given node connected to a
specific network.

However, SLAAC can be problematic when used with servers and VMs. Some operating
systems will not wait for autoconfiguration to complete and some server software
will either fail to start or even fall back to the loopback interface to listen
on when the primary interface is not ready early enough upon startup. Due to
those potential race conditions it is recommended to use static IP configuration
on servers.

.. hint:: **Maintain the SLAAC IP for printers in DNS**

   In order to be easily reachable from clients, the SLAAC IP of every printer
   should be recorded in the DNS zone for nodes and a separate record should
   be maintained in the DNS zone for services pointing to the respective node
   name.

   Using SLAAC for printers spares administrators from the tedious exercise to
   input verbose IPv6 addresses via single button interfaces.

.. hint:: **Use a predictable addressing scheme for servers**

   An IPv6 ``/64`` subnet has room for 18446744073709551616 addresses. The wast
   size of those subnets opens the opportunity to discourage host discovery by
   network scans (there are many pitfalls though, see `RFC 7707`_). In order to
   avoid clustering hosts around likely scanned ranges, one could use a
   cryptographic hash of the hostname as the basis for an IP address.
   Implementations of this method are available as an ansible filter
   (`znerol/ipaddr_hash on galaxy.ansible.com`_) and as a JavaSrcipt isomorphic
   library (`ipaddrhash on npmjs.com`_). For convenience, the fully functional
   example webapp can be found and used at `znerol.github.io/ipaddrhash-js/`_

.. _`RFC 7707`: https://datatracker.ietf.org/doc/html/rfc7707
.. _`znerol/ipaddr_hash on galaxy.ansible.com`: https://galaxy.ansible.com/znerol/ipaddr_hash
.. _`ipaddrhash on npmjs.com`: https://www.npmjs.com/package/ipaddrhash
.. _`znerol.github.io/ipaddrhash-js/`: https://znerol.github.io/ipaddrhash-js/
