# Introduction

Microsoft Azure services can be classified, according to their network-level behavior, as either a public service or a private service.

**Azure private services** are deployed into Virtual Networks (VNets). Azure VNets are isolated networks defined within an Azure region. Users can freely define the address space of their VNets, using public or private (RFC 1918) addresses. Addresses used in a VNet are assumed not to be routable outside of the VNet itself. Therefore, different VNets can use the same address space, or partially overlapping address spaces. Azure services deployed within VNets are assigned IP addresses taken from the VNet’s address space. As such, they are not directly reachable from the internet. The quintessential example of a private service is an Azure Virtual Machine.

**Azure public services** are not deployed into any VNet. They are directly accessible from the internet, over public IP addresses owned by Microsoft and assigned to service instances at provisioning time. Public services can be consumed by any client with an internet connection (provided that the client possesses the required credentials).  Most Azure PaaS services are public, examples of which include Azure SQL Database, Azure Storage and Azure Cosmos DB. 

PaaS services democratized access to advanced IT capabilities that, with more traditional computing paradigms, would be affordable only for very large organizations with enough resources (financial and technical) to acquire and manage the underlying infrastructure. However, the public nature of most PaaS services has soon emerged as a blocker to their adoption in many security-sensitive scenarios.  Microsoft Azure supports three **VNet integration patterns** to make PaaS services only accessible from clients deployed within Azure VNets and not accessible from the internet. Multiple patterns are required because of the architectural differences that exist among different public PaaS services. 

This article describes the three VNet integration patterns currently available on the platform (VNet injection, VNet service endpoints and Private Link) and the relationship between the architecture of a PaaS service and the integration pattern(s) that can be applied to it.

# PaaS architectures and VNet integration patterns

One of the defining attributes of cloud computing is massive horizontal scalability. Cloud services run on large pools of resources (compute, storage, network) managed by a centralized control plane. When a user requests the service, the control plane creates an **instance** of the service for that user and allocates resources in the pool to the instance. Massive scale is achieved by adding resources to the pool when more capacity is needed to meet user demand.

There are two ways a service can allocate resources to instances. 

* A specific set of resources (compute, storage, network) is allocated to one specific instance, and one only, for the duration of that instance’s lifecycle. Services that allocate resources in this way are said to have a **dedicated architecture**, because each resource in the pool is allocated to a single instance at any point in time. In this document, they will be referred to as **dedicated services**.
* A specific set of resources (compute, storage, network) is allocated to more than one service instance, with instances concurrently consuming capacity from that set of resources. Services that allocate resources in this way are said to have a **shared architecture**, because each resource in the pool is shared across multiple instances. In this document, they will be referred to as **shared services**.

Whether a service has a dedicated or a shared architecture is in most cases dictated by the very nature of the service itself. In turn, a service’s architecture dictates the VNet integration pattern(s) that can be applied to make it accessible only from clients deployed within Azure VNets and not accessible from the internet. The next two paragraphs cover the relationship between service architecture (dedicated vs. shared) and VNet integration patterns. 

## VNet integration patterns for dedicated services

Azure Cache for Redis is an example of a dedicated PaaS service. Its high-level architecture is shown in Figure 1 below. When a user provisions an instance of the service, specific resources are allocated to that instance. Instances belonging to other customers use different sets of resources.

> Please note that resources in a pool may be virtualized. Therefore, services with a dedicated architecture do not necessarily assign dedicated *hardware* resources to individual instances.

![Figure 1 - NOT DISPLAYED](/Figures/figure1.png)
*Figure 1. Azure Cache for Redis has a dedicated architecture. Each instance runs on a dedicated set of resources.*

Azure PaaS services with a dedicated architecture can be made private by deploying the resources dedicated to an instance into a VNet belonging to the owner of that instance. This integration pattern is referred to as **VNet injection**. It is described in the section titled "VNet injection".

## VNet integration patterns for shared services

Azure Storage is an example of a shared service. Its high-level architecture is shown in the picture below. Multiple storage accounts belonging to different users (i.e. multiple instances of the storage service) share the same set of compute, storage and network resources (which is referred to as a **storage stamp**).

![Figure 2 - NOT DISPLAYED](/Figures/figure2.png)
*Figure 2. Azure Storage has a shared architecture. Multiple instances belonging to different users share the same compute, storage and network resources.*

Shared services cannot be deployed into any user’s VNet because they run on resources that must be accessible by multiple users. Just like a public web site that must be available to users connecting from multiple private networks (such as their home network or their company’s network), the resources allocated to a shared service instance must be exposed on public, internet-routable IP addresses owned by the service provider. Therefore, VNet injection (introduced in the previous section specific to *dedicated services*) is not a viable integration pattern for shared services. For shared services, two alternative patterns are available: **VNet Service Endpoints** and **Private Link**. These integration patterns are described in sections "VNet service endpoints" and "Private Link", respectively.

# VNet injection

VNet injection is the VNet integration pattern for services whose architecture is based on **dedicated resources** that can be deployed (aka “injected”) into the instance owner’s VNet, as shown in Figure 3 below.

![Figure 3 - NOT DISPLAYED](/Figures/figure3.png)
*Figure 3. VNet injection for dedicated services (Azure Cache for Redis is used as an example). Resources allocated to a service instance can be deployed to a VNet belonging to the instance’s owner and therefore can be exposed on private IP addresses in the VNet’s address space.*

* VNet-injected services are usually deployed to a dedicated subnet that cannot contain any other services (such as Virtual Machines) deployed by the user. The minimum number of required IP addresses in the subnet depends on the service. Please refer to the official documentation of each specific service for details.
* VNet-injected services are exposed over IP addresses that belong to the VNet’s address space. Therefore, at the network layer, these services behave just like Virtual Machines. More specifically, they can 
	* Initiate connections to, and receive connections from, Virtual Machines in the same VNet (or in other VNets connected to it via VNet peering or VNet-2-VNet IPSec tunnels);
	* Initiate connections to, and receive connections from, on-prem hosts over VPN tunnels or Expressroute;
	* Initiate connections to internet-routable IP addresses outside the VNet’s address space by leveraging the [default Source-NAT functionality provided by the VNet](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-outbound-connections);
	* Receive inbound connections from routable IP addresses outside the VNet’s address space when exposed behind a public IP address. The public IP address used by the service is usually a front-end IP address of an Azure External Load Balancer.

Please note that a specific service might not leverage all the capabilities listed above. 

## Implications of VNet injection on network security groups and route tables

The control plane of dedicated services does *not* have a dedicated architecture itself. It manages multiple resources allocated to different instances and users. Therefore, it cannot be injected in any VNets and must rely on endpoints with public IP addresses to interact with the VNet-injected resources it manages. As a result, VNet-injected resources must be able to (see Figure 3 above):

* Initiate outbound connections to public IP addresses associated to control plane endpoints, for example to send notifications to the management layer or to download configuration updates from a storage facility, such as Azure SQL DB or an Azure Storage account; 
* Receive inbound connections from public IP addresses associated to their control plane, for example to be notified about events in their lifecycle (provision, deprovision, apply configuration, etc.).

This is shown in Figure 4 below.

![Figure 4 - NOT DISPLAYED](/Figures/figure4.png)
*Figure 4. VNet-injected services require inbound and outbound connections from/to platform-managed public IP addresses to interact with their control plane.*

The two conditions above are met when the network security groups (NSGs) and the user-defined routes (UDRs) applied to subnets that host VNet-injected resources adhere to the following configuration guidelines:

* Allow inbound connections from the set of public IP addresses/ports used by the control plane. These IPs depend on the service type and on the region where it is deployed. Furthermore, due to the dynamic nature of the cloud, they may vary over time;
* Allow outbound connections to the set of public IP addresses/ports used by the service’s dependencies. These IPs depend on the service type and on the region where it is deployed. Furthermore, due to the dynamic nature of the cloud, they may vary over time;
* Do *not* change the next hop in the VNet’s system route table for traffic destined to the service’s dependencies. Doing so may cause the connections to be source-NATted behind IP addresses different than the ones expected by the service’s dependencies and, therefore, dropped.

Not all the above constraints necessarily apply to all services. For example, some VNet-injected services are compatible with custom routing configurations whereby all traffic destined to public IP addresses outside the VNet's address range is force-tunneled to on-prem over Site-2-Site or Expressroute connections. Please refer to each service’s official documentation for details. 

## Service tags

Because of the requirements and constraints listed in the previous section, defining and maintaining NSGs and UDRs for subnets that host injected services is a complex task. To reduce management overhead for NSGs, **service tags** have been introduced in the platform. [Service tags](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview#service-tags) are monikers for sets of IP addresses assigned to a service’s control plane and dependencies, which can be referenced in NSG rules (see Figure 5). As the mapping between a service tag and the associated set of IPs is automatically managed by the platform, service tags allow users to:

* Define NSG rules for subnets that host VNet-injected services without specifying the set of IP addresses assigned to the services' control plane and dependencies;
* Define NSG rules that do not require any updates when the set of IP addresses assigned to a service’s management layer or dependencies changes.

![Figure 5 - NOT DISPLAYED](/Figures/figure5.png)

*Figure 5. Service tags can be used to reference sets of IP addresses associated to a service’s control plane when defining NSG rules for the subnets that host VNet-injected services (screenshot from Azure management portal).*

# VNet Service Endpoints

VNet service endpoints are a VNet integration pattern that can be applied to select Azure PaaS services with a *shared architecture* to make them accessible only from authorized VNet/subnets. More specifically:

* Each *instance* of a PaaS service that supports VNet service endpoints can be configured to accept connections that originate from select subnets in select VNets only. This configuration is applied to the service-side. It should be noted that VNet service endpoints do not prevent internet clients from connecting to a service’s public endpoint; however, all services that support VNet service endpoints also provide a firewalling feature that allows blocking all connections from the internet (or accepting connections only from known, trusted public IPs);
* In each subnet, service endpoints can be enabled for select service types. For example, a subnet can be configured with service endpoints for Azure SQL DB and Azure Storage. This configuration is applied to the client (VNet) side.

> For the latest information about Azure services that support VNet service endpoints, please refer to the [official documentation](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview). 

> From the perspective of clients in a VNet, service endpoints provide access to a whole service, rather than select instances.

Figure 6 below shows a VNet service endpoint configuration example. UserA’s storage account has been configured to only accept connections from clients in Subnet1 of UserA’s VNet (Figure 6.a) and from clients connecting from the public IP address 5.5.5.5 (Figure 6.b). All other connections are rejected, including connections from clients in other subnets (Figure 6.c) and other public IP addresses (Figure 6.d).

![Figure 6 - NOT DISPLAYED](/Figures/figure6.png)
*Figure 6. Services that support VNet service endpoints provide a “Firewalls and virtual networks” feature whereby each service instance can be configured to be only accessible from select subnets in select VNets and from trusted public IP addresses.*

Two constraints must be considered when working with VNet service endpoints:

* The PaaS service instance and the authorized VNet(s) can be in the same Azure subscription or in different Azure subscriptions. In the latter case, the two subscriptions must trust the same Azure Active Directory (AAD) tenant. Please refer to the [official documentation](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview#provisioning) for additional details;
* For some PaaS services, the service instance and the VNet must be in the same region (or in paired regions). For other services, VNet service endpoints allow access to all instances in all regions. Please refer to the [official documentation](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview#limitations) for additional details.

## Implications of VNet service endpoints on network security groups and route tables

From the perspective of a client deployed in an Azure VNet, configuring a VNet service endpoint for a PaaS service does not change the service’s *public* IP addresses. However, the Azure networking stack facilitates connections to those public IP addresses in such a way that the service can verify whether they originate from authorized subnets. This is done in the following way:

* When a service endpoint is configured in a subnet for a service type in a region, routes for all the public prefixes used by that service type in that region are added to the subnet’s route table, with next hop type set to the special value “VirtualNetworkServiceEndpoint” (see Figure 7 below);
* All the IP packets that match one of the routes with next hop type “VNet service endpoint” are encapsulated, by the Azure network virtualization stack, in outer packets that carry information about the identity of their source VNet/subnet; 
* The routes with next hop type “VirtualNetworkServiceEndpoint” are managed by the platform and cannot be overridden by user-defined routes. 

![Figure 7 - NOT DISPLAYED](/Figures/figure7.png)
*Figure 7. Effective routes for a NIC attached to a subnet for which VNet service endpoints have been configured for the Storage service . The highlighted routes are added by the platform and cover all the public prefixes used by the Storage service in the VNet's region and in its paired region (50 and 46 public prefixes, respectively).*

Network security groups can be used to allow/deny outbound connections to PaaS services over VNet Service endpoints. As VNet service endpoints do not change the **public** IP addresses of PaaS services, service tags (covered in section “Service tags”) can be used in NSG rules.

## VNet service endpoint policies 

When VNet service endpoints for a specific service type are enabled for a subnet, then clients in that subnet have network-level access to **all** instances of that service (in that region or in multiple regions, depending on the specific service) – including instances belonging to other users. This introduces the risk of data exfiltration incidents, whereby malicious actors with access to an organization’s VNet can copy that organization’s data from service instances controlled by the organization to service instances not controlled by the organization (see Figure 8, Left).

**VNet service endpoint policies** have been introduced to address the data exfiltration issue. Service endpoint policies allow VNet owners to control exactly *which instances* (identified by their unique fully qualified domain name (FQDN)) of a service type can be accessed from their VNet via service endpoints (see Figure 8, Right).

![Figure 8 - NOT DISPLAYED](/Figures/figure8.png)
*Figure 8. (Left) VNet service endpoints do not prevent data exfiltration attacks. A malicious actor with access to User B’s VNet can read data from User B’s storage account, copy it to a rogue storage account accessible from the public internet and download it from outside User B organization’s network. (Right) VNet service endpoint policies allow VNet owners to specify which service **instances** (as opposed to service **types**) can be accessed via VNet service endpoints and to prevent data exfiltration incidents.*

VNet service endpoint policies are available only for Azure Storage and only in specific regions. Please refer to the [official documentation](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoint-policies-overview) for the most up-to-date information. 

# Private Link

Private Link is the “V2” integration pattern for services with a *shared architecture*. It addresses the limitations of VNet service endpoints by exposing select PaaS services with a *shared architecture* via private IP addresses belonging to a user VNet’s address space. Private Link, in itself, does not prevent internet clients from accessing the service through its public endpoint. However, all services that support Private Link provide a firewall feature that can be configured to block all connections from the internet (or to only accept connections from known, trusted public IPs). This feature, combined with Private Link, enables users to make their PaaS service instance completely private, i.e. exclusively accessible from authorized VNets over private IP addresses belonging to the VNets’ address spaces.

> Please refer to the [official documentation](https://docs.microsoft.com/en-us/azure/private-link/) for up-to-date information about the services that support Private Link.

![Figure 9 - NOT DISPLAYED](/Figures/figure9.png)
*Figure 9. Private Link is a technology that allows clients in a VNet to consume PaaS services through “private endpoints” with IP addresses belonging to the VNet’s address space. In this example, a VM in User A’s VNet with private IP address 10.57.0.15 accesses User A’s storage account through the private address 10.57.1.5, which belongs to User A VNet’s address space.*

The key advantages offered by Private Link over VNet service endpoints are the following:

* Private Link exposes PaaS instances over private IP addresses belonging to a VNet’s address space, thus removing the need to manage routes and NSG rules for public address ranges;
* Private Link makes it possible for on-premises clients connected via Expressroute private peering or VPN to consume PaaS service instances via a private IP address;
* Private Link allows VNet owners to expose select service instances, as opposed to all instances of a service type in a region, to the VNet, thus addressing by design the data exfiltration issue (see paragraph “VNet service endpoint policies”);
* Private Link does not introduce any limitations about the location of the PaaS service instance and the VNet(s) where it is exposed (i.e. PaaS instances and VNets can be in different Azure regions or geographies);
* Private Link can be used to privately expose to a VNet not only first-party PaaS services, but also 3rd party services running on Azure.

Private Link is a platform-wide functionality, but each PaaS service to which it is applicable must be onboarded. Therefore, while Private Link does provide a more effective approach to securing PaaS services than VNet service endpoints, the two integration patterns are expected to coexist in the short and medium term. 

Private Link is exposed to users through two new Azure resource types: 

* **Private Endpoints** (`Microsoft.Network/PrivateEndpoints`) 
* **Private Link Services** (`Microsoft.Network/PrivateLinkServices`)

They are covered in the following paragraphs.

## Private Endpoints 

To expose a public service instance (such as an Azure storage account or an Azure SQL Database) in a VNet with Private Link, a **private endpoint** resource (of type `Microsoft.Network/PrivateEndpoints`) must be provisioned. A private endpoint resource represents a logical relationship between the public service instance and a NIC attached to the VNet where the service is exposed. The NIC resource is automatically created when the private endpoint is provisioned. Just like any other NIC, the NIC associated to a private endpoint gets an IP address in the address range of the subnet it is attached to. That address becomes the address of the service instance for clients in the VNet (or in remote networks connected via VPN or Expressroute private peering). 

The same public service instance can be referenced by multiple private endpoints in different VNets/subnets, even if they belong to different users/subscriptions or if they have overlapping address spaces, as shown in Figure 10.

![Figure 10  - NOT DISPLAYED](/Figures/figure10.png)
*Figure 10. Multiple users can define service endpoints in their VNet. Each user can only consume their own instances via the private endpoint.*

## Private Link Services

Private Link can also supports exposing 3rd party services deployed on the platform through private endpoints. A 3rd party service exposed with Private Link is referred to as a **private link service**. Any custom service running in a VNet behind a Standard SKU Azure Internal Load Balancer (ILB) can be exposed through a private endpoint in another VNet. The custom service and the VNet can reside in different regions and belong to different subscriptions that trust different Azure Active Directory tenants. 

To create a private link service, a resource of type `Microsoft.Network/PrivateLinkServices` must be provisioned. It represents a logical relationship between the Azure ILB that exposes the 3rd party service and a NIC attached to the same VNet as the ILB. The NIC resource (of type `Microsoft.Network/networkInterfaces`) is automatically created as part of the private link service provisioning process. From the service’s perspective, client connections originate from this NIC.

In the consumer VNet, private link services are exposed in the same way as 1st party ones, i.e. as private endpoints (resources of type `Microsoft.Network/PrivateEndpoints`, see previous paragraph, “Private Endpoints”).

Private Link allows service providers to control their services’ exposure. Access to a Private Link service can be granted to all platform users, restricting to a set of trusted subscriptions, or controlled via Azure RBAC. Also, when an authorized user creates a private endpoint to consume a Private Link service, the service provider must approve the request before traffic from the private endpoint is accepted. Please refer to the [official documentation](https://docs.microsoft.com/en-us/azure/private-link/private-link-service-overview) for additional information. Figure 11 provides an example of a private link service exposed via a private endpoint. 

![Figure 11 - NOT DISPLAYED](/Figures/figure11.png)
*Figure 11. Private Link can be used to expose custom services running behind an Azure ILB to other VNets. On the service side, the services to be exposed are represented as “private link services”. On the consumer side, they are represented as “private endpoints”. Clients in the service consumer’s VNet connect to the service using the private endpoint’s address (10.57.1.8 in this example). In the service provider’s VNet, traffic from clients originates from the private link service’s IP address (172.16.4.4 in this example). The platform takes care of the required address translations.*

## Implications of Private Link on network security groups and route tables

One of the key benefits of Private Link is that traffic to PaaS services goes to IP addresses within the address space of the VNets where private endpoints are defined. Therefore, Private Link completely removes the management overhead for NSG rules and/or UDRs for public IP address ranges that exist with VNet service endpoints.

There are however two caveats that must be considered. Both will be addressed in future releases. 

* Private Link does not currently support NSGs. While subnets containing private endpoints can have NSG associated with them, the rules will not be effective on traffic processed by the private endpoints. Please refer to the [official documentation](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview#limitations) for additional details.
* When a private endpoint is created in a VNet, a platform managed /32 route (with next hop type = “InterfaceEndpoint”) for the private endpoint’s IP address is added to the VNet’s route table (see Figure 12). Traffic that matches this route is encapsulated by the Azure SDN stack and routed to the target PaaS service instance in outer packets. Just like any system route, the /32 route can be overridden by UDRs. Therefore, the next hop for response traffic “from” private endpoints can be customized. For example, it can be routed to network virtual appliances that inspect traffic between PaaS services and their clients.

![Figure 12 - NOT DISPLAYED](/Figures/figure12.png)
*Figure 12. Effective routes for a NIC attached to a subnet where a private endpoint has been defined. A platform managed /32 route with next hop type = “InterfaceEndpoint” is added to the VNet’s route table.*

## DNS integration

Each instance of an Azure public service is assigned, at provisioning time, a unique, publicly resolvable FQDN. When a service has a shared architecture, multiple instances run on the same set of resources and share the same IP address. Azure relies on CNAME records to map the FQDNs of different instances to the same IP address, as shown in Figure 13.

![Figure 13 - NOT DISPLAYED](/Figures/figure13.png)
*Figure 13. CNAME records are used to map to the same public IP address to multiple DNS names assigned to different instances of an Azure public service. In the example, multiple storage accounts are exposed on the public IP address 1.2.3.4, whose canonical name is “blob.xyz.store.core.windows.net”. Both User A and User B use aliases (“usera.blob.core.windows.net” and “userb.blob.core.windows.net”, respectively) to access their storage accounts. Azure Storage uses the alias specified in the HOST header of HTTP requests to map each request to the proper storage account.*

It should be noted that clients **must** reference the FQDN of the instance they want to access when connecting to the service. Services with a shared architecture use the FQDN to map each connection received on the shared IP address to the proper instance. The mechanism whereby an instance’s FQDN is embedded in requests depends on the protocol supported by the service. For example, services that use HTTP, like Azure Storage, map requests to instances based on the FQDN specified in the HTTP requests’ HOST header. In the example of Figure 13, HTTP requests with HOST header “**usera**.blob.core.windows.net” are mapped to User A’s storage account; HTTP requests with HOST header “**userb**.blob.core.windows.net” are mapped to User B’s storage account. Requests submitted using either the endpoint’s IP address (1.2.3.4) or its canonical name (blob.xyz.store.core.windows.net) generate an HTTP error response, because they cannot be mapped to any storage account.

Private Link introduces further complexity in DNS resolution. In the most general scenario, an instance of a service that supports Private Link may be accessed through its “native” public IP *and* through private endpoints in *multiple* VNets at the same time. In both cases (access through public or private endpoint) the FQDN used by clients must remain the same (as explained above); but it must be resolved to different IP addresses, depending on the endpoint (public vs. private) being accessed. More specifically:

* Clients outside VNets (and not connected to VNets with private endpoints through site-to-site VPNs or Expressroute private peering) must resolve the FQDN of the instance they want to access to the service’s public IP address;
* Clients deployed in VNets with a private endpoint must resolve the same FQDN to the IP address assigned to the private endpoint in *their own* VNet. In other words, DNS resolution for service instances exposed over private endpoints must be customized in each VNet.

> It should be noted that Private Link introduces requirements that cannot be addressed by simply running a “split horizon” DNS service. In fact, Private Link requires DNS resolution to change not only based on the client’s location (for example, inside vs. outside an Azure VNet) but also based on whether or not private endpoints have been defined for any given instance.

Microsoft Azure allows users to customize name resolution for service instances associated to private endpoints by adding one more level of indirection in the mapping between instance FQDNs and IP addresses, as detailed below.

* For each service that supports private endpoints, a “privatelink” DNS child zone is defined in the zone assigned to the service. For example, in the case of Azure Blob Storage, the child zone “**privatelink**.blob.core.windows.net” is defined in the parent zone “blob.core.windows.net”.
* For each service *instance* associated to private endpoints, its name in the parent zone is made an alias (via a CNAME record) of the corresponding name in the child zone. For example, if a private endpoint is configured in a VNet for the storage account “usera.blob.core.windows.net”, that name is mapped to “usera.**privatelink**.blob.core.windows.net” via a CNAME record in the parent zone. 
* In Azure public DNS, the names in the “privatelink” zone are then mapped, via additional CNAME records, to the actual, public IPs of the service. Therefore, the additional level of indirection is transparent to internet clients that resolve names via recursive resolvers and the public DNS service, as shown in Figure 14 below.

![Figure 14 - NOT DISPLAYED](/Figures/figure14.png)
*Figure 14. Name resolution for internet clients for the example storage account “usera.blob.core.windows.net”. The additional level of indirection (name “usera.blob.core.windows.net” mapped to “usera.privatelink.blob.core.windows.net”) is transparent to the client.*

VNet owners can map names in the “privatelink” zone to the IP addresses of the corresponding private endpoints in their VNets by means of a custom “privatelink” child zone. The custom zone can be deployed as an [Azure DNS private zone](https://docs.microsoft.com/en-us/azure/dns/private-dns-overview) linked to the VNet, or by means of custom DNS servers. Name resolution for clients within VNets with private endpoints works as shown in Figure 15 below.

> When using custom authoritative DNS servers, they **must** be configured to forward queries for names outside the privatelink zones to the Azure-provided DNS service (168.63.129.16) or to other servers that can resolve names against the public DNS service. This is needed in order for the custom DNS servers to discover whether private endpoints have been defined for a service instance (if yes, the instance FQDN is mapped to a “privatelink” name via a CNAME record; if not, the instance FQDN is mapped via a CNAME record to the canonical name of the service’s public IP address). 

![Figure 15 - NOT DISPLAYED](/Figures/figure15.png)

*Figure 15. Name resolution for clients in an Azure VNet where a private endpoint for the example storage account “usera.blob.core.windows.net” has been defined. The public DNS service is still used in steps 1-3. If the storage account is associated to a private endpoint, step 3 returns its alias in the privatelink zone, which triggers custom resolution based on the zone file maintained in the Azure DNS private zone or on custom DNS servers (step 4).* 

Figure 16 below summarizes the DNS record structure that enables name resolution for the example storage account “usera.blob.core.windows.net” in all cases covered in this paragraph (resolution when no private endpoints have been defined, resolution for internet clients when private endpoints have been defined, resolution for VNet client when private endpoints have been defined).

![Figure 16 - NOT DISPLAYED](/Figures/figure16.png)
*Figure 16. DNS records for the example storage account “usera.blob.core.windows.net” before (A) and after (B, C) defining service endpoints for it.(A) When no private endpoints are associated to the storage account, its name is mapped to the canonical name of the storage stamp’s public IP (one level of indirection, needed to map multiple instance names running on a shared storage stamp to the stamp’s public IP address). (B) When the storage account is associated to a private endpoint, its name “usera.blob.core.windows.net” is mapped via an additional CNAME record to the corresponding name “usera.privatelink.blob.core.windows.net” in the public DNS service. Recursive resolvers that are not authoritative for “privatelink” zones follow the additional level of indirection and return the storage account’s public IP address. (C) Recursive resolvers in an Azure VNet that are authoritative for the “privatelink.blob.core.windows.net” zone query the public DNS service for the storage account name “usera.blob.core.windows.net” too, but then resolve the name “usera.privatelink.blob.core.windows.net” based on an A record in their custom zone file, which provides the address associated to the private endpoint in the VNet.*

### Hybrid scenarios

In hybrid scenarios where on-prem clients consume private endpoints over site-to-site VPNs or Expressroute private peering, customized name resolution for the “privatelink” names must be provided to on-prem clients. 

If Azure DNS is used for “privatelink” zones in Azure VNets, additional DNS servers (authoritative for the “privatelink” zones) must be deployed to support resolution for on-prem clients (Azure DNS cannot be used by on-prem clients). This option requires manual management of the “privatelink” zones on the on-prem DNS servers, and care must be taken to keep the zone file(s) aligned with Azure DNS’s.

If custom DNS servers authoritative for the “privatelink” zones are deployed in Azure VNets (i.e. Azure DNS private zones are not used) on-prem DNS servers should be configured to forward queries for names in the “privatelink” zones to those DNS servers. It is also possible to deploy multiple authoritative DNS servers for the privatelink zones both on-prem and in Azure VNet and use zone transfers to keep them in sync.
