# Site-to-Site VPN connection over ExpressRoute private peering

- [Introduction](#introduction)
- [ExpressRoute](#ExpressRoute)
- [VPN](#VPN)
- [Route Manipulation](#RouteManipulation)
- [Route Symmetry](#RouteSymmetry)
- [Summary](#Summary)

## Introduction 
Site-to-Site VPN Tunnels can be configured over the ExpressRoute circuit that connects your datacenter to Azure. By default, traffic over an ExpressRoute connection is not encrypted, and there are two methods to enable encryption:

+ MACsec, which encrypts data at Layer 2.
+ VPN, which encrypts data in a Site-to-Site manner.

This guideline will help you understand the implications and challenges associated with setting up a VPN Tunnel using the Azure VPN Gateway over an ExpressRoute connection.

## ExpressRoute 
Let's examine the ExpressRoute architecture we are working with:
![](https://github.com/infobozk/ExpressRouteVPN/blob/52df2e7a82b1e01c096bfb91241d0a5ad9167f83/images/ExpressRoute_Setup.png)

Deployment can be done following the guidance on the Azure Docs website. Alternatively, you can use the following Terraform code:
> An existing circuit is assumed, modify where needed.
```
variable "expressroute_circuit" {
 type = string
 description = "Existing ExpressRoute Circuit"
}

resource "azurerm_resource_group" "resourcegroup_demo" {
  name     = "rg-demo"
  location = "West Europe"
}

resource "azurerm_virtual_network" "virtualnetwork_hub" {
  name                = "vnet-hub"
  location            = azurerm_resource_group.resourcegroup_demo.location
  resource_group_name = azurerm_resource_group.resourcegroup_demo.name
  address_space       = ["10.64.0.0/23"]
}

resource "azurerm_subnet" "GatewaySubnet" {
  name                 = "GatewaySubnet"
  resource_group_name  = azurerm_resource_group.resourcegroup_demo.name
  virtual_network_name = azurerm_virtual_network.virtualnetwork_hub.name
  address_prefixes     = ["10.64.0.0/26"]
}

resource "azurerm_public_ip" "pip_ergw" {
  name                = "pip_ergw"
  location            = azurerm_resource_group.resourcegroup_demo.location
  resource_group_name = azurerm_resource_group.resourcegroup_demo.name

  sku               = "Standard"
  allocation_method = "Static"
}

data "azurerm_express_route_circuit" "expressroutecircuit" {
  resource_group_name = "PLACEHOLDER"
  name                = var.expressroute_circuit
}

resource "azurerm_virtual_network_gateway" "ExpressrouteGateway" {
  name                = "ExpressrouteGateway"
  location            = azurerm_resource_group.resourcegroup_demo.location
  resource_group_name = azurerm_resource_group.resourcegroup_demo.name
  type                = "ExpressRoute"
  sku                 = "Standard"

  ip_configuration {
    name                          = "vnetGatewayConfig"
    public_ip_address_id          = azurerm_public_ip.pip_ergw.id
    private_ip_address_allocation = "Dynamic"
    subnet_id                     = azurerm_subnet.GatewaySubnet.id
  }
}

resource azurerm_virtual_network_gateway_connection "expressroute_connection" {
    name                = "expressroute_connection"
    location            = azurerm_resource_group.resourcegroup_demo.location
    resource_group_name = azurerm_resource_group.resourcegroup_demo.name

    type                            = "ExpressRoute"
    virtual_network_gateway_id      = azurerm_virtual_network_gateway.ExpressrouteGateway.id
    express_route_circuit_id        = data.azurerm_express_route_circuit.expressroutecircuit.id
}
```
The ExpressRoute Gateway and On-Prem Routers will exchange routes once the BGP session has been established. This is key to having a VPN Tunnel on top of the ExpressRoute connection: **route manipulation**. It is this mechanism that allows us to direct traffic into the VPN Tunnel, ensuring it does not go outside of it.

The ExpressRoute Gateway will advertise the address space of the Virtual Network in which it resides, along with any peered virtual networks. This is the default behavior of the Virtual Network Gateways. Now, take a look at the on-prem router:

```
R1#show ip bgp
   Network          Next Hop            
*> 10.64.0.0/23    192.168.100.2            
*> 172.16.0.0/16    192.168.100.2                  
```
From the routers in the datacenters, we now understand how to reach those address spaces. Regarding traffic flowing from Azure to the datacenters, it's necessary to advertise the address ranges we wish to communicate over the ExpressRoute connection. For this demonstration, we'll only advertise the loopback interfaces, which results in the following learned routes in Azure:
```
❯ az network express-route
LocPrf    Network             NextHop         Path            Weight
--------  ------------------  --------------  --------------  --------
          10.10.10.1/32       192.168.100.1   65501           0       
          10.20.10.1/32       192.168.200.1   65501           0           
```
These IP addresses will be used as the VPN endpoints.
## VPN
Now, let's examine the VPN architecture layered on top of the ExpressRoute connection:
![](https://github.com/infobozk/ExpressRouteVPN/blob/52df2e7a82b1e01c096bfb91241d0a5ad9167f83/images/VPN_Setup.png)
Important points to note:

+ Utilize the Private IP addresses of the VPN Gateways. These addresses are advertised over the BGP session of the ExpressRoute Circuit.
+ APIPA addresses will be used for BGP peering. While not a requirement, they will be employed in this demonstration.
+ The VPN Gateways will be deployed across Availability Zones in an active-active configuration.

```
resource "azurerm_public_ip" "pip01_vpngateway" {
  name                = "pip01_vpngateway"
  location            = azurerm_resource_group.resourcegroup_demo.location
  resource_group_name = azurerm_resource_group.resourcegroup_demo.name
  zones               = [ "1", "2", "3" ]
  sku                 = "Standard"
  allocation_method   = "Static"
}

resource "azurerm_public_ip" "pip02_vpngateway" {
  name                = "pip02_vpngateway"
  location            = azurerm_resource_group.resourcegroup_demo.location
  resource_group_name = azurerm_resource_group.resourcegroup_demo.name
  zones               = [ "1", "2", "3" ]
  sku                 = "Standard"
  allocation_method   = "Static"
}

 resource "azurerm_virtual_network_gateway" "vpngateway" {
   name                = "vpngateway"
   location            = azurerm_resource_group.resourcegroup_demo.location
   resource_group_name = azurerm_resource_group.resourcegroup_demo.name

   type     = "Vpn"
   vpn_type = "RouteBased"
   sku      = "VpnGw2AZ"
   active_active = true
   enable_bgp    = true

   ip_configuration {
     name                          = "vnetGatewayConfig1"
     public_ip_address_id          = azurerm_public_ip.pip01_vpngateway.id
     private_ip_address_allocation = "Dynamic"
     subnet_id                     = azurerm_subnet.gatewaysubnet_conhub.id
   }
  
   ip_configuration {
     name                          = "vnetGatewayConfig2"
     public_ip_address_id          = azurerm_public_ip.pip02_vpngateway.id
     private_ip_address_allocation = "Dynamic"
     subnet_id                     = azurerm_subnet.gatewaysubnet_conhub.id
   }

   bgp_settings{
     asn                            = "65515"
     peering_addresses {
         ip_configuration_name = "vnetGatewayConfig1"
         apipa_addresses = [ "169.254.21.2"]
     }
     peering_addresses {
         ip_configuration_name = "vnetGatewayConfig2"
         apipa_addresses = [ "169.254.22.2"]
     }
   }
 }

resource "azurerm_local_network_gateway" "datacenter_router01" {
  name                = "datacenter_router1"
  resource_group_name = azurerm_resource_group.resourcegroup_demo.name
  location            = azurerm_resource_group.resourcegroup_demo.location
  gateway_address     = "PLACEHOLDER"

  bgp_settings {
        asn           = "PLACEHOLDER"
        bgp_peering_address = "169.254.21.1"
    }
}

resource "azurerm_local_network_gateway" "datacenter_router02" {
  name                = "connectivityhub-prd-weu-lgw-02"
  resource_group_name = azurerm_resource_group.rg_conhub.name
  location            = azurerm_resource_group.rg_conhub.location
  gateway_address     = "PLACEHOLDER"

  bgp_settings {
        asn           = "PLACEHOLDER"
        bgp_peering_address = "169.254.22.1"
    }
}

 resource "azurerm_virtual_network_gateway_connection" "datacenter01_connection" {
   name                = "datacenter01_connection"
   location            = azurerm_resource_group.resourcegroup_demo.location
   resource_group_name = azurerm_resource_group.resourcegroup_demo.name

   type                       = "IPsec"
   virtual_network_gateway_id = azurerm_virtual_network_gateway.vpngateway.id
   local_network_gateway_id   = azurerm_local_network_gateway.datacenter_01.id

   shared_key = "PLACEHOLDER"
   enable_bgp = true
   local_azure_ip_address_enabled = true

   custom_bgp_addresses {
     primary = "169.254.21.2"
     secondary = "169.254.22.2"
   }
 }

 resource "azurerm_virtual_network_gateway_connection" "datacenter02_connection" {
   name                = "datacenter02_connection"
   location            = azurerm_resource_group.resourcegroup_demo.location
   resource_group_name = azurerm_resource_group.resourcegroup_demo.name

   type                       = "IPsec"
   virtual_network_gateway_id = azurerm_virtual_network_gateway.vpngateway.id
   local_network_gateway_id   = azurerm_local_network_gateway.datacenter_02.id

   shared_key = var.ipsec_psk
   enable_bgp = true
   local_azure_ip_address_enabled = true

   custom_bgp_addresses {
     primary = "169.254.21.2"
     secondary = "169.254.22.2"
   }
 }
```
With the VPN in place, the datacenter routers will now have an additional method for learning about routes from Azure.
```
R1#show ip bgp
   Network          Next Hop            
 10.64.0.0/23    169.254.21.2            
 172.16.0.0/16    169.254.21.2

 10.64.0.0/23    192.168.100.2            
 172.16.0.0/16    192.168.100.2                    
```
Although this output is trimmed, it demonstrates that our routers now have multiple ways to reach the other side. 

+ Both routers are aware of a route to 10.64.0.0/23, with one route going over the ExpressRoute connection (via 192.168.100.2 and 192.168.200.2) and another route going over the VPN connection (via 169.254.21.2 and 168.254.22.2).

## Route Manipulation
We understand that both of our Gateways in Azure will advertise the address spaces of the Virtual Networks in which they are deployed, as well as those of any peered Virtual Networks. We also know that, by default, routers will accept these advertisements and incorporate them into their routing tables.

BGP offers several methods for accepting or denying routes. One such method involves using Prefix Lists and Route Maps:

+ Prefix Lists
```
Define a prefix list for the advertised routes
ip prefix-list Azure-Advertised-Routes seq 10 permit 10.64.0.0/23
ip prefix-list Azure-Advertised-Routes seq 20 permit 172.16.0.0/16
```
+ Route Maps
```
Define a route map to set higher local preference for VPN Gateway
route-map VPN-Gateway-Prefer permit 10
 match ip address prefix-list Azure-Advertised-Routes
 set local-preference 200

Define a route map for other routes (lower preference)
route-map Other-Routes permit 10
 match ip address prefix-list Azure-Advertised-Routes
 set local-preference 100

Apply the route maps to the BGP neighbors
router bgp 65501
 neighbor 169.254.21.2 route-map VPN-Gateway-Prefer in
 neighbor 192.168.100.2 route-map Other-Routes in
```
The strategy here is as follows:

+ The ExpressRoute Gateway and VPN Gateway continue to advertise the same BGP routes over their respective BGP sessions.
+ Through filtering, we accept and integrate the BGP routes from the VPN Gateway into the local router's routing table.
+ BGP also provides the flexibility to select which routes we advertise towards Azure. This means we can choose to only advertise specific network address spaces to the Gateways.
+ It's important to remember that the subnet address space of the Gateway Subnet should continue to be routed over the ExpressRoute connection. Failing to do so could disrupt the VPN Tunnel.
+ Other subnets within the Hub Virtual Network can be managed using the filtering method outlined in the example.

## Route symmetry 
With this setup, traffic will flow symmetrically between Azure and the datacenters. Consider a scenario where the architecture is modified slightly:
![](https://github.com/infobozk/ExpressRouteVPN/blob/52df2e7a82b1e01c096bfb91241d0a5ad9167f83/images/Firewall_Setup.png)

+ An Azure Firewall is introduced into the Hub Network.
+ A User-Defined Route (UDR) directs all traffic from the spokes to the Azure Firewall.

As a result, our traffic flows will change, as illustrated here:
![](https://github.com/infobozk/ExpressRouteVPN/blob/52df2e7a82b1e01c096bfb91241d0a5ad9167f83/images/Firewall_TrafficFlow.png)

We observe that the traffic flow is not symmetrical. Incoming traffic bypasses the Azure Firewall and directly targets the Virtual Machine. Conversely, traffic originating from the Azure Virtual Machine first encounters the Azure Firewall before reaching the VPN Gateway.

To address this, we can assign a Route Table to the GatewaySubnet:

+ All traffic destined for the Spoke Virtual Networks, and passing through the Gateways, will be routed to the Azure Firewall. This ensures that our traffic flow remains symmetric, as shown in the following diagram:
![](https://github.com/infobozk/ExpressRouteVPN/blob/52df2e7a82b1e01c096bfb91241d0a5ad9167f83/images/Firewall_TrafficFlowSymmetric.png)

## Summary
As we conclude this guideline, let's recap the essential aspects of integrating VPN and ExpressRoute for secure and efficient network traffic management between Azure and your datacenters:

+ Integration of ExpressRoute and VPN: We explored how to configure Site-to-Site VPN Tunnels over an ExpressRoute circuit.

+ Route Manipulation Strategies: The guideline emphasized the importance of BGP in route manipulation, using Prefix Lists and 

+ Route Maps to selectively accept and advertise routes, ensuring optimal traffic flow.

+ Maintaining Route Symmetry: We discussed the significance of symmetric routing, especially when introducing components like Azure Firewall. The assignment of a Route Table to the GatewaySubnet was identified as a key step to manage traffic flow effectively.

