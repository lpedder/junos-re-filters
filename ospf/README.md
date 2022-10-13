# OSPF

## General Notes
If you are using OSPF in your network, understand the use case
 * Will your network only use OSPF point-to-point links?
 * Do you need to use broadcast segments and therefore accommodate DR/BDR election?

In the case of a network exclusively using point-to-point links, the destination will always be 224.0.0.5 (AllSPFRouters), according to [RFC2328](https://www.rfc-editor.org/rfc/rfc2328):

>"The IP destination address for the packet is selected as
>follows.  On physical point-to-point networks, the IP
>destination is always set to the address AllSPFRouters."

Therefore the filter can be restricted to only allow this destination addres. This will significantly reduce the available attack surface.
