.. nfdhcpd documentation master file, created by
   sphinx-quickstart on Mon Jan 20 18:25:17 2014.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to nfdhcpd's documentation!
===================================

nfdhcpd is a userspace server written in python and based on NFQUEUE [1].  The
administrator can enable processing of DHCP, NS, RS, DHCPv6 requests on
individual TAP interfaces by injecting nfdhcpd in the processing pipeline for
IP packets dynamically (by mangling the corresponding packet types and redirect
them to the appropriate nfqueue).

The daemon runs on the host and is controlled by manipulating files under its
state directory. Creation of a new file under this directory ("binding file")
instructs the daemon to reply on the requests arriving on the specified TAP
interface.

nfdhcpd vs. dnsmasq
-------------------

a) The service can be activated dynamically, per-interface, by manipulating
iptables accordingly. There is no need to restart the daemon, or edit
(potentially read-only) configuration files, you only need to drop a file
containing the required info under /var/lib/nfdhcpd.

b) There is no interference to existing DHCP servers listening to port
67. Everything happens directly via NFQUEUE.

c) The host doesn't even need to have an IP address on the interfaces
where DHCP replies are served, making it invisible to the VMs. This
may be beneficial from a security perspective. Similarly, it doesn't
matter if the TAP interface is bridged or routed.

d) Binding files provide a TAP-MAC mapping. In other words, requests coming
from unregistered TAP interfaces (without a binding file) are ignored, and
packet processing happens as if nfdhcpd didn't exist in the first place.
Requests coming from a registered device with but with different are considered
as snooping attempts and are dropped.

e) nfdhcpd is written in pure Python and uses scapy for packet
processing. This has proved super-useful when trying to troubleshooting
networking problems in production.

A simple scenario
-----------------

a) nfdhcpd starts. Upon initialization, it creates an NFQUEUE (e.g., 42,
configurable), and listens on it for incoming DHCP requests. It also begins to
watch its state directory, `/var/lib/nfdhcpd` via inotify().

b) A new VM gets created, let's assume its NIC has address mac0, lives on TAP
interface tap0, and is to receive IP address ip0 via DHCP.

c) Someone (e.g., a Ganeti KVM ifup script, or in our case snf-network [2]
creates a new binding file informing nfdhcpd that it is to reply to DHCP
requests from MAC mac0 on TAP interface tap0, and include IP ip0 in the DHCP
reply.

d) The ifup script or the administrator injects nfdhcpd in the processing
pipeline for packets coming from tap0, using iptables:

.. code-block:: console

  # iptables -t mangle -A PREROUTING -i tap0 -m udp -p udp --dport 67 -j NFQUEUE --queue-num 42

e) From now on, whenever a DHCP request is sent out by the VM, the
iptables rule will forward the packet to nfdhcpd, which will consult
its bindings database, find the entry for tap0, verify the source MAC,
and inject a DHCP reply for the corresponding IP address into tap0.

Binding file
------------

A binding file in nfdhcpd's state directory is named after the
physical interface where the daemon is to receive incoming DHCP requests
from, and defines at least the following variables:

* ``INSTANCE``: The instance name related to this inteface

* ``INDEV``: The logical interface where the packet is received on. For
  bridged setups, the bridge interface, e.g., br0. Otherwise, same as
  the file name.

* ``MAC``: The MAC address where the DHCP request must be originating from

* ``IP``: The IPv4 address to be returned in DHCP replies

* ``SUBNET``: The IPv4 subnet to be returned in DHCP replies in CIDR form

* ``GATEWAY``: The IPv4 gateway to be returned in DHCP replies

* ``SUBNET6``: The IPv6 network prefix

* ``GATEWAY6``: The IPv6 network gateway

* ``EUI64``: The IPv6 address of the instance


nfdhcpd.conf
------------

The configuration file for nfdhcp is `/etc/nfdhpcd/nfdhcpd.conf`. Three
sections are defined: general, dhcp, ipv6.

Note that nfdhcpd can run as nobody. This and other options related to
its execution environment are defined in general section.

In the dhcp section we define the options related to DHCP responses.
Specifically:

* ``enable_dhcp`` to globally enable/disable DHCP

* ``server_ip`` a dummy IP that the VMs will as src IP of the response

* ``dhcp_queue`` the a NFQUEUE number to listen on for DHCP requests

| Please not that this queue *must* be used in iptables mangle rule.

* ``nameservers`` IPv4 nameservers to include in DHCP responses

* ``domain`` the domain to serve with the replies (optional)

| If not given the instance's name (hostname) will be used instead.

In the ipv6 section we define the options related to IPv6 responses.  Currently
nfdhcpd supports IPv6 stateless configuration [3]. The instance will get an
auto-generated IPv6 (MAC to eui64) based on the IPv6 prefix exported by Router
Advertisements (O flag set, M flag unset). This kind of RA will make instance
query for nameservers and domain search list via DHCPv6 request.
nfdhcpd, currently and in case of IPv6, is supposed to work on a routed setup
where the instances are not on the same collision domain with the external
router and thus any RA/NA should be served locally. Specifically:

* ``enable_ipv6`` to globally enable/disable IPv6 responses

* ``ra_period`` to define how often nfdhcpd will send RAs to TAPs with IPv6

* ``rs_queue`` the NFQUEUE number to listen on for router solicitations

* ``ns_queue`` the NFQUEUE number to listen on for neighbor solicitations

* ``dhcp_queue`` the NFQUEUE number to listen on for DHCPv6 request

* ``nmeservers`` the IPv6 nameservers

| They can be send using the RDNSS option of the RA [4].
| Since it is not supported by Windows we serve them via DHCPv6 responses

* ``domains`` the domain search list

| If not given the instance's name (hostname) will be used instead.

iptables
--------

In order nfdhcpd to be able to process incoming requests you have to mangle
the corresponding packages. Please note that in case of bridged setup the
kernel understands that the packets are coming from the bridge (logical indev)
and not from the tap (physical indev). Specifically:

* **DHCP**: ``iptables -t mangle -A PREROUTING -i tap+ -p udp --dport 67 -j NFQUEUE --queue-num 42``

* **RS**: ``ip6tables -t mangle -A PREROUTING -i tap+ -p icmpv6 --icmpv6-type router-solicitation -j NFQUEUE --queue-num 43``

* **NS**: ``ip6tables -t mangle -A PREROUTING -i tap+ -p icmpv6 --icmpv6-type neighbour-solicitation -j NFQUEUE --queue-num 44``

* **DHCPv6**: ``ip6tables -t mangle -A PREROUTING -i tap+ -p udp --dport 547 -j NFQUEUE --queue-num 45``

For a bridged setup replace tap+ with br+ in case of DHCP. Using nfdhcpd
for IPv6 in a bridged setup does not make any sense. The above rules are
included in `/etc/ferm/nfdhcpd.ferm` .
In case you use ferm, this file should be included in `/etc/ferm/ferm.conf`.
Otherwise an `rc.local` script can be used to issue those rules upon boot.


debug
-----

A useful way to see the clients registered in nfdhpcd runtime context one can
send SIGUSR1 and see the list in the logfile:

.. code-block:: console

 # kill -SIGUSR1 $(cat /var/run/nfdhcpd/nfdhpcd.pid) && tail -n 100 /var/log/nfdhcpd/nfdhpcd.log


| [1] https://www.wzdftpd.net/redmine/projects/nfqueue-bindings/wiki/
| [2] https://code.grnet.gr/projects/snf-network/
| [3] https://tools.ietf.org/html/rfc4862
| [4] https://tools.ietf.org/html/rfc5006
