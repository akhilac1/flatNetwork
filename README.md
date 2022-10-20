# Design for Cross cluster communication in Dapr

This document is a proposal to enable communication across clusters in a container environment,and discusses the design options to support this behaviour in Dapr enabled clusters.

## Table of Contents
1. [Objective / Scenarios](#objective)
1. [Goals](#goals)
1. [Out of Scope](#out-of-scope)
1. [Requirements](#requirement)
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

## Objective

In Kubernetes there are scenarios where multiple users - for example - dev, QA teams want to use a cluster environment. These can be seggregated via namespaces.
Multi-tenancy is the ability to have multiple distinct namespaces within a single cluster. Quotas and policies can be applied for each namespace and this enables simple solution to seggregate tenants.

Multi-cluster environment is where there are many ``seperate`` clusters that need to communicate. These clusters can exist in a falt network setup where every pod IP is reachable from every other pod. Setting up such clusters is beyind the scope of this proposal. 

Also, multiple clusters could be ``federated`` to allow for communication. 
Both these scenarios are in the scope of this proposal.

The intention of this proposal is to enable Dapr features like actor invocation , service invocation to work seamlessly and in a cohesive manner multi-cluster environments. 

### Common Scenarios for Multicluster
- #### Scenario 1 :  
    We have multiple clusters for security reasons with clear access policies for communication across these clusters. I should be able to securely communicate between the applications running in these clusters. 
- #### Scenario 2 : 
    I have multiple clusters that have distributed applications, that interact with each other to deliver an end-to-end experience. Microservices in one cluster have to constantly interact with microservices in other cluster.
- #### Scenario 3 : 
    I have multiple clusters for high availability. My applications run as multiple copies across these clusters. It is very important that when one cluster fails, the other cluster takes up the traffic, with minimal disruption to the user. I want to be able to define policy on which services should be leveraged as primary or failover.

- #### <b>TBD</b> -- Add more scenarios

These scenarios lead to the following requirements - 
- #### Requirement 1 :
    Applications within a cluster should be able to communicate with applications deployed in other connected clusters, without having to make  changes in the implementation/code. 

- #### Requirement 2 :
    It should be possible to securely communicate across the clusters by using a common root CA to allow mTLS encrypted traffic across services.

- #### Requirement 3 :
    It should be possible to define policies to disambiguate service discovery when multiple instances of the app are running in multiple clusters.

    For example if there are multiple instances of the target app being invoked, user should be able to specify if the local app be given first priority vs. app running on a cluster in the a same region vs. app running on a cluster in the same availability region vs. app running in a different region.
     ![failover scenario](https://user-images.githubusercontent.com/3939554/196892722-fdd08049-1f77-4413-b2ba-05d4f3e2b8ca.png)

- #### Requirement 4 :
    Daprized applications should be able to communicate with the another instance of the target application when a failover happens


### Goals
This document will focus on the following goals :
*	Enable Service Discovery  - Enhance name resolution to function for name resolution across clusters to address [ Listed Scenarios ](#common-scenarios-for-multicluster) to support - 
    
                - Service Invocation across clusters
                - Actor Invocation across clusters
    
*   Enable definition of access policy in a multi-cluster environment


### Out of Scope
*   Networking setup and reachability between clusters is  a pre-requisite. 
*   Setting up mTLS across multiple clusters is not in scope for this proposal 
* Ingress configuration is not in scope for this proposal

### Requirement

| Priority | Requirement | Description|
| :---: | :--- | :---|
|P0 | Setup Configuration | Enable configuration to be setup for an app to  discover applications in other clusters. |
|P0 | Name Resolution Building Block| Name Resolution should continue to function as-is in case of cluster and non-cluster environments
| P0 | Enhance Dapr  name-resolution components to support cross-cluster communication | Name resolution components should support cross-cluster service discovery in adherence with policies specified |
|P0 | Service invocation across clusters | Support Service Invocation across cross-cluster services with no impact on SDK / API|
|TBD | Tracing | Ensure trace can be correlated across cross-cluster communication|
|TBD | Control plane service | Control plane service to manage cross cluster access policy |
|TBD | Actor communication across clusters | Support actors to invoke calls across clusters |

** P0 requirements are targetted for design and implementation for 1.10, while we continue to evolve the design and implementation for rest of the scenarios.


### Constraints

Following are the constraints to ensure the new solution does not break existing implementation :

1. There should be no breaking SDK changes to enable multi-cluster communication

1. There should be no assumptions on the cluster topology. When a new configuration is added, App should be able to consume the new configuration without needing a restart  

1. Tracing should be supported across clusters

## Concepts

### What is Service Discovery

In the world of microservices implemented using Dapr, destination services are referred to by their registered app name. The process of translating the registered app name to a internal gRPC port of the dapr side car is part of service discovery in Dapr.

This is achieved via name resolution building block and components.

Depending on the name resolution component user chooses, a registry is maintained to keep track of services and their IP Addresses. Each name resolution component has different strategy to build this registry and to keep it updated. This strategy also impacts the ability of the name resolution component to scale.

Also the name resolution component is responsible to health checking and keeping the service registry updated. This enables the service to work without having to build and maintain awareness of about other services in the network.

### How does service discovery work in Dapr today?

Name resolution is a building block in Dapr. Loading a name resolution component is mandatory in both self-hosted and kubernetes modes.
This building block enables -
1. app registry with the chosen or default name resolution component and 
2. lookup for resolving the app name to target IP as part of service discovery.

Dapr supports multiple components for name resolution. Each component has a distinct strategy to resolve and provide the IP address for a given name. Following table documents a high level strategy for each supported component and its ability to support multi-cluster environments.

| Component | Domain Name Resolution Strategy | Supports Multi-Cluster
| :---: | :--- | :---: 
mDNS| Supports name resolution in a small network by issuing multicast UDP query to all hosts in the network, expecting the host to identify itself. This is not scalable across large networks and works well for Dapr self-hosted mode | No
Hashicorp Consul | Provides DNS registry and query interface | Yes
Kubernetes | Configured as default name-resolution component in kubernetes mode by dapr. DNS pod and service are deployed on the cluster | No

Below is an illustration of Name Resolution in a single cluster standalone mode for [Service invocation quickstart](https://docs.dapr.io/getting-started/quickstarts/serviceinvocation-quickstart/) in standalone mode.

![Dapr Name Resolution Flow](https://user-images.githubusercontent.com/3939554/193627883-a24bd7c0-447a-470f-96ec-930a0df4861c.png)

## Strategies for Multi-cluster communication in a container environment

There are multiple strategies to support cross cluster communication :

1. ### Gateways - 
    Clusters are managed via independent service meshes and operate in unified trust domain (aka. share a common root certificate/signing certificate). An ingress gateway is configured to accept trusted traffic across these clusters. 
    The downside of this approach is additional overhead of configuring the networks to allow cross-cluster traffic.

1. ### Flat Network - 
    All the pods in each cluster have a non-overlapping IP address. Clusters can communicate via VPN tunnels, to ensure all pods are able to communicate with all other pods on the network. 
    
    This scenario has scalability issues, overhead of managing non-overlapping IP addresses across clusters,and puts all the clusters in a common fault-domain. 

3. ### Selective End-point discovery - 
    This approach enables limited visibility of services in the cluster, for each application.   Sidecar for a pod is configured with list of end points that a service wants to talk to. 
    
    If the pod is in another cluster, that is assumed to be reachable via an ingress gateway, then service discovery provides the IP address of the ingress gateway instead. When a pod routes traffic to a ingress gateway, the ingress gateway uses ServerName Indication ([SNI](https://en.wikipedia.org/wiki/Split-horizon_DNS)) to know where the traffic is destined. This approach does not require a flat network. We will discuss the design details for this approach in the subsequent sections.

### Proposed Approach for Multi-Cluster Communication in Dapr

As discussed above, there are several approaches to achieve cross-cluster communication in Dapr. 

![Proposed implementation](https://user-images.githubusercontent.com/3939554/196891073-4c3c3304-a110-4801-9ca6-9d88c94efe6e.png)

1. Dapr should support mTLS traffic between dapr side cars running different clusters configured via any of the [listed strategies](#strategies-for-multi-cluster-communication-in-a-container-environment).

2. Dapr name resolution should enable service discovery in all the above scenarios. 

3. Dapr should provide a standard approach to setup and run applications in a multi-cluster environment, irrespective of the network provider.

4. Users should be able to specify required configuration via a configuration store. 

5. Dapr name resolution component should be enhanced to refer to the configuration store to retrieve cluster configuation information to help in resolving to the right target address. 

    For example, if the orderProcessor application in the above example is running in three different clusters, checkout application should get the right destination order processor application details and dapr side car should be able to communicate with the obtained IP endpoint. 

1. In the initial phase, we propose that the Configuration of allowed endpoints per application for each side-car be setup via Configuration Store. 

    Since configuration store does not support 'SET' API to add configuration, this would need to be setup directly in the configuration store.

## Additional References
1. [SNI](https://cheapsslsecurity.com/blog/simplified-what-is-sni-server-name-indication-how-does-it-work/)