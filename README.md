# JunOS Filter Elements
Junos Filter Elements by Protocol

These filter elements aim to provide RFC-referenced and compliant filtering. They would typically be used to build a Routing Engine (RE) Filter and be applied inbound on the lo0 interface. They use the principle of least privilege e.g. the filter elements are as specific as possible

Feedback and corrections welcome, please submit a pull request or open an issue to discuss

## Things to decide First
When constucting a loopback filter there are two questions that need to be answered first:
 * Will your router need to process IP Options?
 * Will your router need to process fragments?

These packets are punted to the RE and a possible attack vector. If you don't need these packets on your router, discard them first.

### IP Options

If your network runs RSVP on an older version of JunOS <15.2 (or uses multiple vendors) then perhaps it relies upon the IP Options Router Alert bit. Upon receipt of an packet with this bit set, the router will punt it to the RE, opening a possible attack vector. If you run RSVP the chances are it now uses RFC2961 refresh reduction which prohibits the use of the Router Alert bit in bundled messages.

If you are sure your network does not need IP Options processing, then deal with this first.
 * [Discard Term for IPv4 Options](options/inet.conf)

### Fragments

In a properly dimensioned network, fragmentation should not occur to the router control plane.

In IPv4, the first-fragment is easily identified:
 * More Fragments bit set to 1
 * Fragment Offset field set to 0

If you are not doing fragment processing, then there is no point in completing further layer 3/4 matches on this packet. It needs to go, and accepting it will only take up reassembly buffer space. It should be immediately dropped.

Subsequent IPv4 Fragments will have:
 * Fragment Offset field non-zero

Because the RE filters are simply bitwise matching on the packet header, there is a theoretical possibility subsequent fragment payloads could match a firewall filter term. So to be absolutely sure, any packet with a non-zero fragment offset should be immediately discarded before further terms are evaluated.

 * [Discard Term for IPv4 Fragments](fragments/inet.conf)

With IPv6, because all fragments are placed in fragment headers, they can be identified easily:
 * next-header value set to fragment (44).

For IPv6 RE filtering, it is safer to accept on next-header. The next-header match cannot be bypassed and typically everything else gets dropped at the final term. So if you have a long list of next-header accept terms and a discard term at the end, IPv6 fragments will be safely dropped here.

## General Approach

Creating RE filters is by no means easy. They are *stateless* filters which can be a challenge for engineers used to working with stateful firewall policies. This task requires
 * Understanding to *packet level* of protocols in use on the network
 * Knowledge of all network addressing
 * Knowledge of how edge filtering must work in conjunction with RE filters for effective protection

There are some common pitfalls, so the general approach taken here in building the elements is detailed here.

### Use of Prefix-Lists and Apply-Paths
When constructing a firewall filter it can quickly get complex. If you use abstraction to multiple levels this can make it more difficult to read. For this reason, the examples here don't refer to prefix lists or other filters. This makes what is allowed for each protocol clearer. There are other problems with using prefix lists:
 * The temptation to re-use them can mean your filters aren't as specific as they could be
 * Using JunOS apply-path wildcards in prefix lists can open your filter much wider than you expect

In the case of a commonly used apply-path wildcard such as this:
```
    prefix-list router-ipv4 {
        apply-path "interfaces <*> unit <*> family inet address <*>";
    }
```
You might think that the result will be the /32 addresses of each interface. But this actually takes the full subnet of the interfaces as you can see below:
```
user@router> show configuration policy-options prefix-list router-ipv4 | display inheritance
##
## apply-path was expanded to:
##     172.16.24.0/27;
##     172.16.24.128/29;
##     203.0.113.0/24;
##
apply-path "interfaces <*> unit <*> family inet address <*>";
```
This can be danegerous if you are connected to a large peering LAN for example. All the peers would potentially be permitted by your filters referring to this prefix-list!

Better to know exactly what you are permitting and where you are permitting it from. This is particularly true when using GTSM on BGP peers, which relies on correct filtering based on TTL values.

### Always Match Source IPs

There are only two things that should normally be permitted to the router from anywhere:
 * ICMP for essential operations
 * Traceroute (UDP unix-style)

Everything else should be matched on source IPs.

### Never Match on Port

When it comes to matching port numbers, JunOS allows you use three match conditions:
 * **port** - Which matches on either source or destination port
 * **source-port** - Matches only the source port
 * **destination-port** - Matches only the destination port

A common pitfall is to match only using **port**. The problem here is best explained with an example. In the following filter term:
```
    term accept-bgp {
        from {
            source-address {
                198.51.0.1/32;
            }
            protocol tcp;
            port bgp;
        }
        then {
            count accept-bgp;
            accept;
        }
    }
```
The remote BGP peer 198.51.0.1 would be able to establish a TCP connection to this router on any destination port simply by setting the source port to 179. This would allow access to port 22 (SSH) for example, exposing a potential attack surface.

For BGP, the filter should be two terms. Each term matches for one direction as either end can initiate a BGP session. You can see this in the elements provided below.

### Also Match on Destination IP If Possible

Following the principle of least privilege, if there is a way to further restrict a filter term to a known destination IP then it should be used. This is particularly important if your network uses L3VPNs. Because VPN customers may be using the same source IP addresses you have permitted, the destination address should also be matched on a filter term.

### Restrict TTLs Where Specified in RFCs

If an RFC states a protocol MUST set a TTL or hop-limit value for a packet type, then this can help to further limit access in a filter term. For example in IPv6 Neighbor Discovery, the hop-limit value is set to 255. This should be matched in the filter term for ND packets.

## Protocols
Here are individual filter elements for each protocol, with references provided to RFCs and documentation. Some protocols have a specific README file that explains the rationale.

* [BGP](bgp)
* [OSPF](ospf)
* [VRRP](vrrp)
* [ICMP](icmp)
* [RSVP](rsvp)
* [LDP](ldp)
* [BFD](bfd)

## Management Protocols
Individual filter elements for each management protocol

* [SSH](ssh)
* [SNMP](snmp)
* [NTP](ntp)
* [Traceroute](traceroute)
