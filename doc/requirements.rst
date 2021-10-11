Requirements
************

This guide is not a tutorial. Readers should be familiar with IPv4 and IPv6
addressing and terminology.

Throughout this guide we assume that the internet uplink provides native IPv6
connectivity with a prefix sized large enough such that it can be subdivided
into multiple ``/64`` networks. Respectable ISPs will assign static IPv6
prefixes between a ``/60`` (min) and ``/48`` (max) either by default or on
request.

Deployments stuck with an IPv4 only uplink or with dynamic / dysfunctional IPv6
connectivity may choose to upgrade to static IPv6 with the help of a `tunnel
broker service`_.

.. _tunnel broker service: https://en.wikipedia.org/wiki/List_of_IPv6_tunnel_brokers

