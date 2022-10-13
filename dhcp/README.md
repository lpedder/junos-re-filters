# DHCP

## General Notes
DHCP on JunOS is not a very nice protocol to protect. According to the Juniper [documentation](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/subscriber-management-dhcp-firewall-filter.html) there are a couple of things going on:

 * When an FPC receives any DHCP packet, it will change both source and destination ports to 68.
A
 * These packets are re-encapsulated in new UDP headers by jdhcpd and sent to the RE

Therefore, firewall filters matching DHCP packets need to cater for ports seen by the FPCs *and* the REs.

There is also something unusual going on with DHCP relay. When the RE relays a request to a DHCP server, the source and destination ports leaving the RE are both 67. The DHCP server will reply with the same ports. When this reply arrives on the FPCs, it is again translated to 68 for both source and destination port. Therefore both ports must be alloed in these filters.

Additionally, Juniper [states](https://www.juniper.net/documentation/us/en/software/junos/dhcp/topics/topic-map/dhcp-relay-agent-security-devices.html) that:

>After establishing the initial lease on the IP address, the DHCP client and the DHCP server use unicast transmission to negotiate lease renewal or release. The DHCP relay agent “snoops” on all of the packets unicast between the client and the server that pass through the router (or switch) to determine when the lease for this client has expired or been released. This process is referred to as lease shadowing or passive snooping.

In general, only use this filter if you know you are using DHCP or DHCP relay. The filter should be monitored to ensure terms are being correcty matched.
