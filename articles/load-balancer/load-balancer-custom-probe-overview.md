---
title: Azure Load Balancer health probes
description: Learn about the different types of health probes and configuration for Azure Load Balancer.
author: mbender-ms
ms.service: load-balancer
ms.topic: conceptual
ms.date: 05/05/2023
ms.author: mbender
ms.custom: template-concept, engagement-fy23
---

# Azure Load Balancer health probes

Azure Load Balancer rules require a health probe to detect the endpoint status. The configuration of the health probe and probe responses determines which backend pool instances receive new connections. Use health probes to detect the failure of an application. Generate a custom response to a health probe. Use the health probe for flow control to manage load or planned downtime. When a health probe fails, the load balancer stops sending new connections to the respective unhealthy instance. Outbound connectivity isn't affected, only inbound.

Health probes support multiple protocols. The availability of a specific health probe protocol varies by Load Balancer SKU. Additionally, the behavior of the service varies by Load Balancer SKU as shown in this table:

| | Standard SKU | Basic SKU |
| --- | --- | --- |
| **[Probe protocol](#probe-protocol)** | TCP, HTTP, HTTPS | TCP, HTTP |
| **[Probe down behavior](#probe-down-behavior)** | All probes down, all TCP flows continue. | All probes down, all TCP flows expire. | 

## Probe configuration

Health probe configuration consists of the following elements:

| Health Probe configuration | Details |
| --- | --- | 
| Protocol | Protocol of health probe. This is the protocol type you would like the health probe to leverage. Available options are: TCP, HTTP, HTTPS |
| Port | Port of the health probe. The destination port you would like the health probe to use when it connects to the virtual machine to check the virtual machine's health status. You must ensure that the virtual machine is also listening on this port (that is, the port is open). |
| Interval (seconds) | Interval of health probe. The amount of time (in seconds) between consecutive health check attempts to the virtual machine |

## Probe protocol

The protocol used by the health probe can be configured to one of the following options: TCP, HTTP, HTTPS.

| Scenario | TCP probe | HTTP/HTTPS probe |
| --- | --- | --- | 
| Overview | TCP probes initiate a connection by performing a three-way open TCP handshake with the defined port. TCP probes terminate a connection with a four-way close TCP handshake. | HTTP and HTTPS issue an HTTP GET with the specified path. Both of these probes support relative paths for the HTTP GET. HTTPS probes are the same as HTTP probes with the addition of a Transport Layer Security (TLS).   HTTP / HTTPS probes can be useful to implement your own logic to remove instances from load balancer if the probe port is also the listener for the service. |
| Probe failure behavior | A TCP probe fails when: 1. The TCP listener on the instance doesn't respond at all during the timeout period.  A probe is marked down based on the number of timed-out probe requests, which were configured to go unanswered before marking down the probe. 2. The probe receives a TCP reset from the instance. | An HTTP/HTTPS probe fails when: 1. Probe endpoint returns an HTTP response code other than 200 (for example, 403, 404, or 500). 2.  Probe endpoint doesn't respond at all during the minimum of the probe interval and 30-second timeout period. Multiple probe requests might go unanswered before the probe gets marked as not running and until the sum of all timeout intervals has been reached. 3. Probe endpoint closes the connection via a TCP reset.
| Probe up behavior | TCP health probes are considered healthy and mark the backend endpoint as healthy when: 1. The health probe is successful once after the VM boots. 2. Any backend endpoint that has achieved a healthy state is eligible for receiving new flows.  | The health probe is marked up when the instance responds with an HTTP status 200 within the timeout period. HTTP/HTTPS health probes are considered healthy and mark the backend endpoint as healthy when: 1. The health probe is successful once after the VM boots. 2. Any backend endpoint that has achieved a healthy state is eligible for receiving new flows.

> [!NOTE] 
> The HTTPS probe requires the use of certificates based that have a minimum signature hash of SHA256 in the entire chain.

## Probe down behavior
| Scenario | TCP connections | UDP datagrams |
| --- | --- | --- | 
| Single instance probes down |  New TCP connections succeed to remaining healthy backend endpoint. Established TCP connections to this backend endpoint continue. |   Existing UDP flows move to another healthy instance in the backend pool.|
| All instances probe down | No new flows are sent to the backend pool. Standard Load Balancer allows established TCP flows to continue given that a backend pool has more than one backend instance.  Basic Load Balancer terminates all existing TCP flows to the backend pool. |  All existing UDP flows terminate. |

## Probe interval

The interval value determines how frequently the health probe checks for a response from your backend pool instances. If the health probe fails, your backend pool instances are immediately marked as unhealthy. On the next healthy probe up, the health probe marks your backend pool instances as healthy. The health probe attempts to check the configured health probe port every 15 seconds by default but can be explicitly set to another value.

For example, if a health probe set to 5 seconds. The time at which a probe is sent isn't synchronized with when your application may change state. The total time it takes for your health probe to reflect your application state can fall into one of the two following scenarios:

1. If your application produces a time-out response just before the next probe arrives, the detection of the events will take 5 seconds plus the duration of the application time-out when the probe arrives. You can assume the detection to take slightly over 5 seconds.
2. If your application produces a time-out response just after the next probe arrives, the detection of the events won't begin until the probe arrives and times out, plus another 5 seconds. You can assume the detection to take just under 10 seconds.

For this example, once detection has occurred, the platform takes a small amount of time to react to the change. The reaction depends on:
* When the application changes state
* When the change is detected
* When the next health probe is sent
* When the detection has been communicated across the platform 

Assume the reaction to a time-out response takes a minimum of 5 seconds and a maximum of 10 seconds to react to the change.
This example is provided to illustrate what is taking place. It's not possible to forecast an exact duration beyond the guidance in the example.

## Probe source IP address

Load Balancer uses a distributed probing service for its internal health model. The probing service resides on each host where VMs and can be programmed on-demand to generate health probes per the customer's configuration. The health probe traffic is directly between the probing service that generates the health probe and the customer VM. All IPv4 Load Balancer health probes originate from the IP address 168.63.129.16 as their source. IPv6 probes use a [link-local address](https://www.wikipedia.org/wiki/Link-local_address) as their source.

The **AzureLoadBalancer** service tag identifies this source IP address in your [network security groups](../virtual-network/network-security-groups-overview.md) and permits health probe traffic by default.

In addition to load balancer health probes, the [following operations use this IP address](../virtual-network/what-is-ip-address-168-63-129-16.md):

* Enables the VM Agent to communicate with the platform to signal it is in a “Ready” state

* Enables communication with the DNS virtual server to provide filtered name resolution to customers that don't define custom DNS servers.  This filtering ensures that customers can only resolve the hostnames of their deployment.
* Enables the VM to obtain a dynamic IP address from the DHCP service in Azure.

## Design guidance

* Health probes are used to make your service resilient scalable. A misconfiguration can affect the availability and scalability of your service. Review this entire document and consider what the effect to your scenario is when the probe response is up or down. Consider how the probe response affects the availability of your application.

* When you design the health model for your application, probe a port on a backend endpoint that reflects the health of the instance **and** the application service. The application port and the probe port aren't required to be the same. In some scenarios, it may be desirable for the probe port to be different than the port your application uses.

* It can be useful for your application to generate a health probe response, and signal the load balancer whether your instance should receive new connections. You can manipulate the probe response to throttle delivery of new connections to an instance by failing the health probe. You can prepare for maintenance of your application and initiate draining of connections to your application. A [probe down](#probe-down-behavior) signal always allows TCP flows to continue until idle timeout or connection closure in a Standard Load Balancer. 

* For a UDP load-balanced application, generate a custom health probe signal from the backend endpoint. Use either TCP, HTTP, or HTTPS for the health probe that matches the corresponding listener.

* [HA Ports load-balancing rule](load-balancer-ha-ports-overview.md) with [Standard Load Balancer](./load-balancer-overview.md). All ports are load balanced and a single health probe response must reflect the status of the entire instance.

* Don't translate or proxy a health probe through the instance that receives the health probe to another instance in your virtual network. This configuration can lead to cascading failures in your scenario. For example: A set of third-party appliances is deployed in the backend pool of a load balancer to provide scale and redundancy for the appliances. The health probe is configured to probe a port that the third-party appliance proxies or translates to other virtual machines behind the appliance. If you probe the same port used to translate or proxy requests to the other virtual machines behind the appliance, any probe response from a single virtual machine marks down the appliance. This configuration can lead to a cascading failure of the application. The trigger can be an intermittent probe failure that causes the load balancer to mark down the appliance instance. This action can disable your application. Probe the health of the appliance itself. The selection of the probe to determine the health signal is an important consideration for network virtual appliances (NVA) scenarios. Consult your application vendor for the appropriate health signal is for such scenarios.

* If you don't allow the [source IP](#probe-source-ip-address) of the probe in your firewall policies, the health probe fails as it is unable to reach your instance.  In turn, Load Balancer marks down your instance due to the health probe failure.  This misconfiguration can cause your load balanced application scenario to fail.

* For Load Balancer's health probe to mark up your instance, you **must** allow this IP address in any Azure [network security groups](../virtual-network/network-security-groups-overview.md) and local firewall policies.  By default, every network security group includes the [service tag](../virtual-network/network-security-groups-overview.md#service-tags) AzureLoadBalancer to permit health probe traffic.

* To test a health probe failure or mark down an individual instance, use a [network security group](../virtual-network/network-security-groups-overview.md) to explicitly block the health probe. Create an NSG rule to block the destination port or [source IP](#probe-source-ip-address) to simulate the failure of a probe.

* Don't configure your virtual network with the Microsoft owned IP address range that contains 168.63.129.16. The configuration collides with the IP address of the health probe and can cause your scenario to fail.

* If you have multiple interfaces configured in your virtual machine, ensure you respond to the probe on the interface you received it on. You may need to source network address translate this address in the VM on a per interface basis.

* Don't enable [TCP timestamps](https://tools.ietf.org/html/rfc1323). TCP timestamps can cause health probes to fail due to the VM's guest OS TCP stack dropping TCP packets. The dropped packets can cause the load balancer to mark the endpoint as down. TCP timestamps are routinely enabled by default on security hardened VM images and must be disabled.

* Note that a probe definition is not mandatory or checked for when using Azure PowerShell, Azure CLI, Templates or API. Probe validation tests are only done when using the Azure Portal.

* If the health probe fluctuates, the load balancer waits longer before it puts the backend endpoint back in the healthy state. This extra wait time protects the user and the infrastructure and is an intentional policy.

* Ensure your virtual machine instances are running. The health probe will probe all running instances in the backend pool. If an instance is stopped, it will not be probed until it has been started again.

## Monitoring

Public and internal [Standard Load Balancer](./load-balancer-overview.md) expose per endpoint and backend endpoint health probe status through [Azure Monitor](./monitor-load-balancer.md). Other Azure services or partner applications can consume these metrics.

Azure Monitor logs aren't available for use with Basic Load Balancer.

## Limitations

* HTTPS probes don't support mutual authentication with a client certificate.

* You should assume health probes fail when TCP timestamps are enabled.

* A Basic SKU load balancer health probe isn't supported with a virtual machine scale set.

* HTTP probes don't support probing on the following ports due to security concerns: 19, 21, 25, 70, 110, 119, 143, 220, 993. 

## Next steps

- Learn more about [Standard Load Balancer](./load-balancer-overview.md)
- Learn [how to manage health probes](../load-balancer/manage-probes-how-to.md)
- [Get started creating a public load balancer in Resource Manager by using PowerShell](quickstart-load-balancer-standard-public-powershell.md)
- [REST API for health probes](/rest/api/load-balancer/loadbalancerprobes/)
