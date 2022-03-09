# Azure Virtual WAN VPN Gateway High Availability Design

Azure Virtual WAN (VWAN) VPN Site-to-Site Gateways run in Active-Active mode.  This means for a given Gateway, Instance 0 (IN_0) and Instance 1 (IN_1) are both actively processing control plane, management plane, and data plane traffic.  This behavior is different from the default behavior of non-VWAN VPN Virtual Network Gateways which is Active-Standby mode.  For VWAN VPN Gateway, each instance IN_0 and IN_1 will have minimally: 

1.	A Public IP address:  used for IPSec tunnel termination and Azure management
2.	A BGP Private IP address:  used for BGP peering with remote VPN site

From a session establishment point of view:

1.	Azure IPSec process can be both initiator and responder, but never responder only.
2.	Azure BGP speaker can act as client or server for BGP TCP Port 179 session (non-APIPA scenario)

To achieve a VWAN high available design, it is recommended that a remote VPN site have two tunnels to Azure VWAN VPN Gateway, one to IN_0 and one to IN_1.  This remote site is configured as a “VPN Site”, which equates to one connection resource.  Both Azure VPN Gateway instances IN_0 and IN_1 will establish S2S VPN tunnel with the remote VPN site.  Note the remote VPN site could be a on-premises datacenter or office, or it could represent a remote VPN appliance sitting in private cloud, including a Azure VNET.

## Configuration Options
There are two configuration categories:  BGP-based and static-based.  
### Configuration 1:  BGP-based configuration
For BGP-based configuration, the design includes two IPSec tunnels to each remote VPN site.  The two IPSec tunnels terminate on each VPN GW instance on the Azure side.  BGP sessions are established over the IPSec tunnels.  Each BGP session is associated with each VPN GW instance on Azure side.  On the remote VPN site, it is recommended the BGP address be associated with a loopback interface, which is always up and decoupled from the IPSec tunnel.  This is an active-active configuration, and in normal operation, traffic from Azure to remote site will be ECMP load balanced across the two IPSec tunnels. 

![BGP-AA-Normal](/Diagrams/BGP-Config1-Normal.png)

Should one of the two instances go out of service, either planned or unplanned, that tunnel path will be marked down and traffic will fail over to the other available instance.  The amount of time depends on whether the outage is planned, and whether ECMP is enabled on the remote side and BGP timers.  Because the assumption in this architecture is two load-shared active-active tunnels, the IPSec tunnel and the BGP Private IP address DO NOT fail over from IN_0 to IN_1, or vice versa.  The second path is expected to pick up all the flows.

![BGP-AA-Failure](/Diagrams/BGP-Config1-Failure.png)

For diversity, a second remote VPN GW may be added at the remote site.  Azure VPN GW would see this second remote GW as another VPN site, another logical connection, with two IPSec tunnels built from each instance IN_0 and IN_1.  Two additional BGP sessions would be built to this second BGP speaker address.    

## Configuration 2:  Static-based configuration 
With static based configuration, both a dual tunnel and a single tunnel configuration are possible:
### Dual Tunnel Configuration
With the dual tunnel configuration, there are two IPSec tunnels formed to each instance IN_0 and IN_1.  The VPN GW will have static definitions to the remote network prefixes, and these networks will be propagated to the Azure fabric so VNETs peered with the vHub will have reachability to these prefixes via the VPN gateway instances as the next hop.  Traffic from Azure to the remote site will be ECMP load shared across the two tunnels.  Similarly, the remote VPN GW will have static definitions to the Azure prefixes via the 2 tunnels.  This configuration is active-active, and in normal operation, traffic from Azure to remote site will be ECMP load balanced across the two tunnels. 

![Static-Dual-Normal](/Diagrams/Static-Config2-Dual-Normal.png)

During failure, the behavior is similar to the BGP example described previously.  The Public IP and the IPSec tunnel associated with the failed instance DO NOT fail over, as it is assumed the other tunnel will continue to process all traffic.

![Static-Dual-Failure](/Diagrams/Static-Config2-Dual-Failure.png)

For diversity, a second remote VPN GW may be added at the remote site.  Azure VPN GW would see this second remote GW as another VPN site, another logical connection, with two IPSec tunnels built from each instance IN_0 and IN_1.  Static routes would be defined to the neighbors prefixes, respectively.  

### Single Tunnel Configuration
In the single tunnel configuration, a single IPSec tunnel is built between one Azure VPN GW Instance and a Remote VPN GW.  This tunnel carries 100% of the traffic.  

![Static-Single-Normal](/Diagrams/Static-Config2-Single-Normal.png)

Should this single tunnel or the instance supporting the tunnel go down during planned or unplanned outage, the Public IP address associated with that instance fails over to the other instance, as does the IPSec tunnel.  From the perspective of the remote VPN site, its single IPSec tunnel is reestablished to the same remote peer.  As the tunnel is rebuilt, user sessions will be re-established across the new tunnel, which may or may not be seamless depending on the application.  

When the failed instance recovers, the Public IP and the tunnel will go back to that instance, causing the user sessions to be re-established again.

![Static-Single-Failure](/Diagrams/Static-Config2-Single-Failure.png)

For diversity and added redundancy, a second remote VPN GW may be added at the remote site with another single tunnel.  This tunnel would be built between the other Azure VPN GW instance.  This way, only 50% of the client sessions would be impacted during any single instance failure, or any single remote VPN GW failure.  

Yet another option would be to add a second IPSec tunnel, but terminate on the same remote VPN GW, but on a different tunnel interface.



