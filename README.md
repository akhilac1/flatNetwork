# Design for Cross cluster communication in Dapr

This document introduces the motivation for enabling communication across clusters in a container environemnt,and discusses the design options to support this behaviour in Dapr enabled clusters.

## Table of Contents
1. [Introduction](#introduction)
1. [Scenarios](#common-scenarios-for-multicluster)
1. [Key Considerations](#key-considerations-across-multi-cluster-flat-network)
1. [Goals](#goals)
1. [Out of Scope](#out-of-scope)
1. [Requirement](#requirement)
1. [Design Constraints](#constraints)
1. [Some concepts to understand the design better](#concepts)

   - [Difference between multi-tenancy and multi-cluster](#difference-between-multi-tenancy-and-multi-cluster)
   - [What is Service Discovery](#what-is-service-discovery)
   - [How does name resolution work in Dapr today?](#how-does-name-resolution-work-in-dapr-today)
1. [Strategies for Multi-cluster communication in a container environment](#strategies-for-multi-cluster-communication-in-a-container-environment)

    - [Gateways](#gateways)
    - [Flat Network](#flat-network)
    - [Selective End Point Discovery](#selective-end-point-discovery)
1. [Proposed Design for Multi-Cluster Communication in Dapr](#proposed-design-for-multi-cluster-communication-in-dapr)
    - [per Cluster](#option-1---manage-configuration-globally-per-cluster)
    - [per App](#option-2---manage-configuration-locally-per-app)
1. [Additional References](#additional-references)

## Introduction
<b>TO-DO</b> Add why cross cluster communication is important. Get customer scenario examples adn how they are limited by lack of this feature

<!-- ** Draft With mainstream Kubernetes adoption increasing many-fold, multiple clusters is now a common scenario. This could be due to many reasons including but not limited to scalability, geo-fencing requirements needing specific applications to reside in specific clusters, multi-cloud scenarios, segregation of networks within the same provider for security reasons etc. -->

The intention of this proposal is to enable Dapr features – actor invocation , service invocation to work seamlessly and in a cohesive manner across single or multi-cluster environments. We will approach this in two phases - 
1. Phase 1 - Design and Proof of concept for Service Invocation 
2. Phase 2 - Design and Proof of concept for Actor Invocation.

### Common Scenarios for Multicluster
- #### Scenario 1 :  I have multiple clusters for security reasons. My organization wanted clearly seggregated boundaries and communication across these clusters is only via access controlled APIs
- #### Scenario 2 : I have multiple clusters that have distributed applications, that interact with each other to deliver an end-to-end experience. Microservices in one cluster have to constantly interact with microservices in other clusters.
- #### Scenario 3 : I have multiple clusters for high availability. My applications run as multiple copies across these clusters. It is very important that when one cluster fails, the other cluster takes up the traffic, with minimal disruption to the user

### Key considerations across multi-cluster Flat network
1. 'Security' - Do we need independent fault domains or will the clusters operate as a unified trust domain?
1. <TBD>

### Goals
This document will focus on the following goals :
*	Enable Service Discovery  - Enhance name resolution to function for name resolution across clusters to address [ Listed Scenarios ](#common-scenarios-for-multicluster) to support - 
    
                - Service Invocation across clusters
                - Actor Invocation across clusters
    
*	Enable Tracing across clusters
*	``< TO-DO > ``   Any other goals 

### Out of Scope
*   Networking setup and reachability between clusters is  a pre-requisite. 
* Ingress configuration where service mesh is involved is not in scope for this proposal

### Requirement

| Priority | Requirement | Description|
| :---: | :--- | :---|
|P0 | Ingest Configuration | Enable configuration to be ingested for app to  discover allowed applications in other clusters. |
|P0 | Service invocation across clusters | Support Service Invocation across cross-cluster services|
|TBD | Actor communication across clusters | Support actors to invoke calls across clusters |
|P0 | Tracing | Enable single view of tracing for cross-cluster communication scenario|
| TBD | Enhance Dapr Building Block to support name-resolution for cross-cluster communication |Existing name resolution components like Consul support cross-cluster service discovery.We need to identify if any new API are required to enable Dapr sidecar to consume this capabililty |
<!--|TBD | Control plan service | Control plane service to enable cross cluster communication |
-->



### Constraints

Following are the constraints to ensure the new solution do not break existing setup :

1. There should be No SDK changes to enable multi-cluster communication

2. There should be no assumptions on the cluster topology. When a new configuration is added, App should be able to consume the new configuration without needing a restart  

## Concepts
### Difference between multi-tenancy and multi-cluster

In Kubernetes there are scenarios where multiple users - for example - dev, QA teams want to use a cluster environment. These can be seggregated via namespaces.
Multi-tenancy is the abilit to have multiple distinct namespaces within a single cluster. Quotas and policies can be applied for each namespace and this enables simple solution to seggregate tenants.

Multi-cluster environment is where there are many ``seperate`` clusters that need to communicate. These clusters are ``federated`` to allow for communication. This is the scenario that we intend to enable in this document.

### What is Service Discovery

Depending on the domain name resolution component user chooses, a registry is maintained to keep track of services and their IP Addresses. Each name resolution component has different strategy to build this registry and to keep it updated. This strategy also impacts the ability of the name resolution component to scale.

Also the name resolution component is responsible to health checking and keeping the service registry updated. This enables the service to work without having to build and maintain awareness of about other services in the network.

### How does name resolution work in Dapr today?

Name resolution is a building block in Dapr. Dapr supports multiple components for name resolution. Each component has a distinct strategy to resolve and provide the IP address for a given name. Following table documents a high level strategy for each supported component and its ability to support multi-cluster environments.

| Component | Domain Name Resolution Strategy | Supports Multi-Cluster
| :---: | :--- | :---: 
mDNS| Supports name resolution in a small network by issuing multicast UDP query to all hosts in the network, expecting the host to identify itself. This is not scalable across large networks and works well for Dapr self-hosted mode | No
Hashicorp Consul | Supports Multi-cluster federation via Mesh-gateway | Yes
Kubernetes | *TO-DO* | *TO-DO*
dns | *TO-DO* | *TO-DO*

Deepdive into Name Resolution in a single cluster standalone mode - <WIP>

![Dapr Name Resolution Flow](https://user-images.githubusercontent.com/3939554/193627883-a24bd7c0-447a-470f-96ec-930a0df4861c.png)

<!-- 
Proposed cross cluster communication -

To achieve multi-cluster communication aka. support service invocation across clusters, we proposed two steps :

1. Enable optional control plane service - 'clusterResolutionService' to be the source for service discovery for applications/services wunning to other connected clusters. 
We discuss the design options to enable this service via dapr annotations and cluster discovery.
2. Enhance nameResolution to respond transparently with information about apps running in other clusters.
3. Enhance service invocation [ including tests ] to work across clusters
-->
<!-- 
### What does federation of clusters mean ? 

Kubernetes multi-cluster environment can be configured in several ways :
1. Multiple clusters within a single host
1. Multiple clusters across multiple hosts within the same datacenter
1. Multiplease clusters in different regions from a single cloud provider
1. Multiple clusters across multiple clouds distributed across multiple regions

Irrespective of the above complexity, multi-cluster environment is managed via 2 strategies :
1. Federated Method
1. Network centric method

### Federated cluster management
Clusters are managed from a centralized location. The focus is on enabling multiple clusters to behave as a single large cluster. Applications are unaware of where they are running.

Kubefed comprises of a federation control plane that runs on the host federated cluster 

## Network centric cluster management
-->
## Strategies for Multi-cluster communication in a container environment
Generally, in a multi-cluster environment, clusters are isolated and remain independent. Applications are independent of the clusters. A service mesh is deployed to manage network connectivity between clusters. 
Service mesh is a fairly mature solution and runs large production clusters. e.g : Istio, Linkerd 

There are multiple strategies to support cross cluster traffic routing :

1. ### Gateways - 
    Clusters are managed via independent service meshes and operate in unified trust domain (aka. share a common root certificate/signing certificate). An ingress gateway is configured to accept trusted traffic across these clusters. 
    The downside of this approach is additional overhead of configuring the networks to allow cross-cluster traffic.

1. ### Flat Network - 
    All the pods in each cluster have a non-overlapping IP address. Clusters can communicate via VPN tunnels, to ensure all pods are able to communicate with all other pods on the network. This is not a common production scenario and has scalability issues, as one common networking control plane has to scale to manage this super-cluster.
    It also puts all the clusters in a common fault-domain and goes against the initial goal of seggregating clusters to seggregate fault domains.

3. ### Selective End-point discovery - 
    Sidecar for a pod is configured with list of end points that a service wants to talk to. This could be a service in the same cluster or in a different cluster. If the pod is in another cluster, that is assumed to be reachable via an ingress gateway, then service discovery provides the IP address of the ingress gateway instead. When a pod routes traffic to a ingress gateway, the ingress gateway uses ServerName Indication ([SNI](https://en.wikipedia.org/wiki/Split-horizon_DNS)) to know where the traffic is destined. This approach does not require a flat network. We will discuss the design details for this approach in the subsequent sections.

### Proposed Design for Multi-Cluster Communication in Dapr

As discussed above, there are several approaches to achieve cross-cluster communication in Dapr. Based on the pros, cons and maturity of the approaches, we propose Dapr to support cross-cluster commounication via [<b><u>Selective end point discovery </u></b>](#selective-end-point-discovery)approach

There are to options to enable selecective end-point discovery for an App in a multi-cluster environment in Dapr
1. #### Option 1 - Manage configuration globally [per Cluster]
    Having a global view of the allowed application at a cluster level has following benefits - 
    - simplifies the cluster management
    - provides a single pane of view for allowed application
    - provides a single point of reference for troubleshooting communication issues

    To manage configuration of allowed endpoints for each application at the cluster level, we would need an admin capability in Dapr. This is currently not supported and hence we defer this option for future when such a capability is supported in Dapr.

1. #### Option 2 - Manage Configuration Locally [per App] 

    In the initial phase, we propose that the Configuration of allowed endpoints per application for each side-car be setup via Configuration Store. We propose a new API for setting up the configuration data to support cross-cluster communication. The propose an opinionated format for data to be saved in the configuration store, to ensure consistency.

We propose managing the configuration per App as the recommended approach to enable cross cluster communication in Dapr.

#### <b>TO-DO</b> Proposed proto structure for cluster configuration data
#### <b>TO-DO</b> Illustration on new nameresolution for cross cluster

<!-- 
Cluster Resolution Service :
1. Local cache on each cluster to look up the location of each app. 
1. Local cache is co-ordinated via placement service.
1. Each app on startup looks for annotation on cluster info
1. If annotation is present it indicates that the app will be part of a specific cluster. So the app also updates its information with the placement service. Details that are of interest are exactly same as the nameresolution. We will add two methods to the client interface :
    - RegisterApp
    - DiscoverApp

1. ~~One placement service will be designated as the master/leader and will perform heartbeat and broadcast any health changes to the other placement services in the joined clusters.~~ 
1. Placement service will maintain a local cache with info about app endpoint info including cluster name
2. If the app does not belong to a cluster, it will be tagged to a default cluster

~~As illustrated in the sequence diagram,service invocation relies on the ability to locate the endpoint that can provide the desired service. All through service invocation flow, the involved client and server IPs are reachable. We proceed with the same underlying assumption and focus on discoverability of these services for service invocation.~~

~~A new control plane service - Cluster Resolution Service [CRS] is proposed to be embedded into Dapr Placement Service. the objective of Dapr Placement Service is to detect the~~

![DaprNameResolution_0 1](https://user-images.githubusercontent.com/3939554/193641131-a1efce31-5dae-48eb-a02c-f3fbaa152519.png)

-->
## Additional References
1. [SNI](https://cheapsslsecurity.com/blog/simplified-what-is-sni-server-name-indication-how-does-it-work/)