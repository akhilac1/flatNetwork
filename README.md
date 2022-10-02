# Design for Cross cluster communication in Dapr

This document introduces the motivation for enabling communication across clusters in a container environemnt,and discusses the design options to support this behaviour in Dapr enabled clusters. We also cover the implementation aspects for the Dapr Building Block.

<TOC>

## Introduction
<TO-DO> Add why cross cluster communication is important. Get customer scenario examples adn how they are limited by lack of this feature

** Draft With mainstream Kubernetes adoption increasing many-fold, multiple clusters is now a common scenario. This could be due to many reasons including but not limited to scalability, geo-fencing requirements needing specific applications to reside in specific clusters, multi-cloud scenarios, segregation of networks within the same provider for security reasons etc.
The intention of this proposal is to enable Dapr features – actor invocation , service invocation to work seamlessly and in a cohesive manner across single or multi-cluster environments.

## Goals
This document will focus on the following goals :
*	Service Discovery
    *	Name Resolution
        *	Enhance name resolution to function for name resolution across clusters
    
    *	Service Invocation across clusters
    *	Actor Invocation across clusters
    
*	Tracing across clusters
*	``< TO-DO > ``   Any other goals 

## Out of Scope
*   Networking setup and reachability between clusters is  a pre-requisite. This is a problem in the infrstructure setup domain and will not be addressed in this proposal. We will proceed with the assumption that the clusters intended to communicate are reachable.

## Requirement

| Priority | Requirement | Description|
| :---: | :--- | :---|
|TBD | Ingest Configuration | Allow *admin* user to upload cluster configuration to Dapr Control-Plane service |
|TBD | Control plan service | Control plane service to enable cross cluster communication |
|TBD | * Dapr Building Block to support cross-cluster config and communication* |This item needs further analysis - will we enhance name resolution or create a seperate building block|
|TBD | Service invocation across clusters | Support Service Invocation across cross-cluster services|
|TBD | Actor communication across clusters | Support actors to invoke calls across clusters |
|TBD | Tracing | Enable single view of tracing for cross-cluster communication scenario|



## Constraints

Following are the constraints to ensure the new solution do not break existing setup :

1. No SDK changes to enable multi-cluster communication

2. No assumptions on the cluster topology to be pre-populated. When a new cluster is added or topology is changed, DAPR control plane should not need a restart   

## Difference between multi-tenancy and multi-cluster

In Kubernetes there are scenarios where multiple users - for example - dev, QA teams want to use a cluster environment. These can be seggregated via namespaces.
Multi-tenancy is the abilit to have multiple distinct namespaces within a single cluster. Quotas and policies can be applied for each namespace and this enables simple solution to seggregate tenants.

Multi-cluster environment is where there are many ``seperate`` clusters that need to communicate. These clusters are ``federated`` to allow for communication. This is the scenario that we intend to enable in this document.

## What is Service Discovery

Depending on the domain name resolution component user chooses, a registry is maintained to keep track of services and their IP Addresses. Each name resolution component has different strategy to build this registry and to keep it updated. This strategy also impacts the ability of the name resolution component to scale.

Also the name resolution component is responsible to health checking and keeping the service registry updated. This enables the service to work without having to build and maintain awareness of about other services in the network.

## How does name resolution work in Dapr today?

Name resolution is a building block in Dapr. Dapr supports multiple components for name resolution. Each component has a distinct strategy to resolve and provide the IP address for a given name. Following table documents a high level strategy for each supported component and its ability to support multi-cluster environments.

| Component | Domain Name Resolution Strategy | Supports Multi-Cluster
| :---: | :--- | :---: 
mDNS| Supports name resolution in a small network by issuing multicast UDP query to all hosts in the network, expecting the host to identify itself. This is not scalable across large networks and works well for Dapr self-hosted mode | No
Hashicorp Consul | Supports Multi-cluster federation via Mesh-gateway | Yes
Kubernetes | *TO-DO* | *TO-DO*
dns | *TO-DO* | *TO-DO*

Deepdive into Name Resolution - <WIP>

![Dapr Name Resolution Flow](.\DaprOrderProcessor_nameresolution_flow.jpg)
