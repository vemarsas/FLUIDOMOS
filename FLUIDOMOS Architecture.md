# **THE FLUIDOMOS ARCHITECTURE**

## 

# Introduction

The **Fluidomos Architecture** describes how to migrate the Delis Domotic Building Cloud Management Platform into microservices runnable across a multi-clustered environment improved by Fluidos.

In-depth, it provides a structural representation of the following:

## 

## A classification for the new containerized services provided by Fluidomos

This document describes a basic classification of DELIS software components and how to migrate them into containerized microservices or applications to comply with the FLUIDOS Computing Continuum operations. It also gives structural guidelines for building software images characterizing Kubernetes deployments.

## The multi-clusters topology

This defines the clusters' topology for implementing Computing Continuum serving Edge controllers. It also defines peering, consumers, and providers of resources when services offloading events occur.

## Services deployment guidelines

Defines which and where services are initially deployed inside the FLUIDOS multi-cluster boundaries. 

## Service offloading guidelines

Define which and where services can be offloaded across the clusters using FLUIDOS.

We start by representing the current DELIS system architecture followed by the new Fluidomos design rethink regarding FLUIDOS and LIQO technologies.

# Delis Architecture

In the current section, we show a diagram to give a brief representation of DELIS architecture followed by a simple description of the DELIS system and its main software components:

<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/delis_architecture.png"><br>
    The current DELIS architecture 
<p>

Briefly, DELIS is a Cloud Platform designed to build, manage, and maintain domotic systems. In this scenario, it offers a web dashboard to execute all the commonly used actions for configuring domotic devices to be managed, acting on them, monitoring the state, and so on.

DELIS software components shown in the previous diagram are:

1. User interface components to provide the Dashboard web application. They are:  
* NGNIX HTTP server to run and provide web access to the DELIS Dashboard  
* DELIS dashboard itself, a web application built using the Ruby on Rails framework  
* SideKiq (used by the Dashboard), a library for efficiently managing background jobs on Ruby  
* REDIS and PostgreSQL, the application in memory and storage databases  
2. Web Service, for interfacing sensors and actuators managed by the Edge controllers:  
   * DELIS provides different web services for interfacing with different branded technologies such as SonOFF, Zway, etc.. In-depth, all web services are developed using Python and a custom framework developed by Vemar.  The framework is the result of Vemar's research in the domotic system. It gives all primitives and abstractions representing how to interface and interact with the domotic devices. By using the framework developers can easily write **Custom Plugins** interfacing with different branded software and  protocols at the edge level

   * Mongo DB is used to store sensors/actuator data

3. Hardware components, like sensors, and actuators directly connected to edge controllers, ones  equipped with branded software to communicate with them

   

# The Fluidomos Architecture: Delis migration on Fluidos

The new FLUIDOMOS architecture, which is derived from the previous regarding DELIS, is shown in the following diagram. In short:

* It redesigns software components in terms of Kubernetes deployments  
* It redesigns the execution environment in terms of Kubernetes clusters,  powered by FLUIDOS and LIQO  
* It organizes the hardware equipment under new boundaries views focused on the physical  location of the domotic units and buildings

<p align="center">
    <img src="https://github.com/vemarsas/FLUIDOMOS/blob/main/resources/fluidomos_architecture.jpg"><br>
    The new FLUIDOMOS architecture
<p>

The architecture diagram can be evaluated from three separate points of view, identifying three different boundaries: Hardware, Services, and the Fluidos Computing Continuum. Each boundary represents an isolated level involving different technologies and skills. 

Communication among different DELIS services translated into Kubernetes deployments is possible through HTTP interfaces. Above all, thanks to Fluoidos, service execution, and orchestration are spanned over Edge and Cloud resources through advertisement/agreement procedures.

## 

## The Hardware Boundary

The first, shown on the left, concerns the hardware equipment. Both hardware equipment with its related branded software are substantially unchanged compared to the previous one by Delis. This boundary describes how typically an edge controller manages sensors and actuators of a home automation system to define the domotic unit to manage. In the new Fluidomos vision, many domotic units can be realized for different places or buildings, which are in turn organized in different geographical areas. As shown using “abstraction” arrows, different areas are respectively managed,  by edge clusters associated with them. A single edge cluster is itself composed of edge nodes representing the edge controller inside domotic units located in a geographical area.

## 

## Service Boundary

It represents a basic model to facilitate the migration of Delis software components into service must be orchestrated and deployed in Kubernetes clusters. The new Fluidomos services should be:

* Off-lodable by Fluidos  
* Deployable into Fluidomos clusters

To facilitate the developments, services are organized and composed by using the following structure:

* Fluidomos Services Images  
  * Python Web Services for the different brands. They are software images built using  the following Delis components:  
    * DELIS Framework (the same one described before under Delis architecture.)  
    * Custom Plugin to interface branded protocols  
  * The Dashboard application image. The same one described before under Delis architecture (but increased with a few features described later)  
* Common Image

All the software images are exposed as Kubernetes deployments. They are initially deployed in the cluster and/or can be offloaded across clusters as shown in the figure

## 

## Fluidos Computing Continuum Boundary

Placed on the bottom-right of the figure, it represents the new computing abstract environment based on a multi-cluster topology that can be orchestrated and managed by Fluidos. The abstract environment is mapped on many K3s clusters: one for each Area defined in the hardware boundaries. The architecture identifies this type of k3s cluster with the name of **Edge Cluster** because each node is defined using a k3s agent running inside the Edge controller.  Moreover, a specialized cluster, called **VPS Cluster**, is designed to have multiple  VPS nodes (each k3s agent runs on a cloud virtual private server) and is used to provide resources where offload Edge Clusters’ services. 

In brief:

1. Edge clusters are defined for each Regional Area and contain their respective Ege Nodes  
2. Services regarding sensor/actuator as well as the web dashboard application are initially deployed into the Edge Cluster and:  
   1. They can be replicated into nodes of the same Edge Cluster where they are been initially deployed  
   2. They can be offloaded into the VPS cluster (which acts as a resource provider) by using the fluidos peering, off-loading, flavors, and solvers

# Cloud DB

The persistent storage engines, used by Fluidomos services, are provided by external Cloud Services. Currently, this choice is due to the complexity of managing stateful applications in Kubernetes environments. However, the new Fluidomos web Dashboard, using the edge file system, will manage the persistence of a minimum set of data necessary to always guarantee the operation of the system (but with reduced functionalities). So that, even in case of unavailability of Internet access (determining the unavailability of the Cloud DB) the overall service, running at the edge level, will be available for users.

Moreover, the Postgress Physical Replication between Cloud and Edge nodes is actually under evaluation to guarantee data are always accessible at the edge level even when Cloud access is unavailable. Physical replication of data will only involve data partitions, designed for each edge node so that each edge node will have its data at the local level. Since this solution is currently under development, its representation in the architecture diagram has been intentionally omitted.

The Vemar Team also includes evaluating the use of Longhorn storage at the Kubernetes level in the future.


