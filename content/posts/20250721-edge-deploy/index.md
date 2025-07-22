---
title: "Edge Deploy"
description: "On Edge one click deploy"
summary: "On Edge one click deploy"
categories: ["Deployment"]
tags: ["Deployment", "IoT"]
##externalUrl: ""
date: 2025-07-21
draft: false
authors:
  - Roque
---

One of the biggest challenges with moving the MES system into the cloud is how can we interface with external systems that must exist near the shopfloor. Today we will take a look at how Critical Manufacturing has solved this issue.

## Overview

Let's dive into one of simplest to understand use cases of the need for an edge application.

It is common for the MES to control machine recipes or machine setups. The question is then, with your MES in the cloud does this work?

![Cloud MES with Recipe](https://image.j-roque.com/posts/20250721-edge-deploy/cloudrecipe.jpg)

And what about this?

![Cloud MES with Recipe World map](https://image.j-roque.com/posts/20250721-edge-deploy/cloudrecipe1.jpg)

Most people would say yes to the first image and no to the second. 

We have an intuitive awareness that if we are trying to access an application or server that is in the other side of the world we will pay a latency cost. In actual fact both images may be the same, it all depends on the type of service that is hired. Most providers offer regional or multi regional hosting, that try to mitigate this issue. 

---

An anecdotal instance that I have seen, was a customer complaining that his system suddenly had got very slow. We did a full investigation on, from the database to the application, and everything seemed ok. When we started checking the network times to response, we noticed that they had worsen in the 10x range. When the customer asked his cloud provider, he was informed that that the nearest datacenter was down for maintenance and they had relocated the service to the nearest datacenter. Well, the original datacenter was already with high latency, when they moved it even further away, it was the difference between an acceptable response time and a response time that is now impacting the operator speed and system usability.

---

Nevertheless, even when we look at use cases where the datacenter distance is not impacting performance, we have use cases where the machine has particular constraints.

![Cloud MES with Recipe Time to Response](https://image.j-roque.com/posts/20250721-edge-deploy/cloudrecipettr.png)

This is also a fairly common scenario. Machines with high speed require process information, validation and acknowledgement to be in the sub second range. Adding the business logic to actually perform control with latency, this can be very hard to achieve with cloud deployments.

## Trade-Offs

We can have systems that have **high data volume**, **high frequency** or **tight time constraints**. We can also have systems that due to **security** reasons must only operate with a local connection.

Systems with large payloads, or that have a high message frequency are going to be more affected by latency, these systems are prime candidates for a decentralized approach.

For CM Connect IoT, which is an application focused on interfacing with third party systems like machines, the ability to be closer to the system we are interfacing with is crucial.

The questions now becomes, how can we have an **application that is near but that also keeps the advantages of the Cloud**.

## Edge Control

The Connect IoT main application, the Automation Manager is very flexible. It can run on Edge or on Cloud, it can be installed in Linux or Windows. It can be explicitly or implicitly installed, let's try and understand what that means.

It can be explicitly deployed via the `CM Devops Center`, or it can be deployed with a `One Click Deploy`. The One Click Deploy is an implicit installation via a middleware component, solely focused on Automation Manager deployment, the Automation Manager Controller (AMC).

The AMC will query the MES for what managers does it need to deploy and deploy them. It can run in the MES cluster or in an edge cluster.

![AMC Deploy](https://image.j-roque.com/posts/20250721-edge-deploy/amcdeploy.png)

The Automation Manager can then spun up all the controllers and drivers it requires. 

![AMC Deploy Managers](https://image.j-roque.com/posts/20250721-edge-deploy/amcdeploymanagers.jpg)

You can see that we can have a very distributed setup where we have several Automation Managers Controllers, deployed in different clusters and then each AMC with several Automation Managers.

## Edge Control - Red Hat's Microshift

One of the questions that customers have regarding containers, particularly edge clusters is about the effort of maintaining not just one cluster with the MES, but the effort of also having clusters on edge and how to maintain both.

This also connects with the need to have a low footprint solution for edge. It must have a low footprint both for running and deploying but also for maintaining.

In order to achieve this, CM provides a virtual machine with `Red Hat's Microshift`. 

Microshift is
- `Lightweight`
- `Kubernetes Compatible`
- `Easy to Maintain & Update`
- `Secure & Reliable`
- `Part of the Red Hat Ecosystem`

Of course, the **customer can always deploy in his own kubernetes managed cluster**, this is a more user friendly and low effort approach. Another interesting alternative can be the use of stretched clusters, where a single Kubernetes cluster is deployed across multiple geographically separated data centers (or availability zones) but function as one logical cluster. There are also proprietary variations on this like [Amazon EKS Hybrid Nodes](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-overview.html). One competing approach to Red Hat's Microshift is the [AKS Edge Essentials](https://learn.microsoft.com/en-us/azure/aks/aksarc/aks-edge-overview).

{{< alert "circle-info" >}}
**Info:** The Microshift virtual machine is currently licensed via Red Hat.
{{< /alert >}}

### Use Case - Microshift with Automation Manager Controller

The process all starts with CM creating an ISO image, and making it available with all that you need to start and install. This image will contain Microshift and a set of custom tools that CM provides to make the user life easier.

![Iso deploy](https://image.j-roque.com/posts/20250721-edge-deploy/iso.jpg)

The user can then install and deploy the virtual machine.

![Deploy Virtual Machine](https://image.j-roque.com/posts/20250721-edge-deploy/deployvm.jpg)

This is where the interesting part starts, the user is then prompted to connect and authenticate with CM.

![CM Connect](https://image.j-roque.com/posts/20250721-edge-deploy/cmconnect.jpg)

He will then register and enroll the cluster in a CM infrastructure. This means CM is now aware of this cluster. 

![Cluster Enrollment](https://image.j-roque.com/posts/20250721-edge-deploy/enrollcluster.jpg)

It will then deploy an `infrastructure agent`. This agent will allow the `CM Devops Center` to not only be aware that the cluster exists, but also what is it's current state and to allow, by user action to deploy additional components.

![Deploy Agent](https://image.j-roque.com/posts/20250721-edge-deploy/deployagent.jpg)

Let's see an example of an actual deployment on my machine.

<video controls width="100%">
  <source src="https://image.j-roque.com/posts/20250721-edge-deploy/deployvm.mp4" type="video/mp4">
</video>

We have now an installed remote cluster. Let's now deploy our AMC.

![Deploy AMC](https://image.j-roque.com/posts/20250721-edge-deploy/deployautomationmanagercontroller.jpg)

<video controls width="100%">
  <source src="https://image.j-roque.com/posts/20250721-edge-deploy/deployamc.mp4" type="video/mp4">
</video>

### Use Case - Automation Manager Controller Deploy Automation Manager

The AMC will continually query the MES system for **changes in the list of Automation Managers** to deploy.

![AMC Query](https://image.j-roque.com/posts/20250721-edge-deploy/amcquery.jpg)

When the user checks the button to change a manager to ready to deploy. The AMC will detect that change and **spin up a new Automation Manager**.

![AMC New Manager](https://image.j-roque.com/posts/20250721-edge-deploy/amcnewmanager.jpg)

The AMC is now ready to deploy new managers or undeploy existing managers. The Automation Managers also have in **built controls where they can have stop and start controls for all their internal components**, like controllers and drivers.

![AMC New Manager Deployed](https://image.j-roque.com/posts/20250721-edge-deploy/amcmanagerdeployed.jpg)

---

Let's see an example of this. I have two clusters, my `mes-summit-edge cluster` in the cloud with the **MES** installation and a `cluster on my machine` with the **AMC**. 

In my MES system I have an Automation Manager with a single controller with a single driver. What we can see is that we have deployed the AMC in the remote cluster and using just our MES we are able to `One Click Deploy` our Automation Manager, with a TCP-IP driver and a controller.

<video controls width="100%">
  <source src="https://image.j-roque.com/posts/20250721-edge-deploy/AMC-DeploymentManager.mp4" type="video/mp4">
</video>

We can also perform other actions like undeploy or mass deploy of several Automation Managers.

<video controls width="100%">
  <source src="https://image.j-roque.com/posts/20250721-edge-deploy/AMC-MassDeploy.mp4" type="video/mp4">
</video>

## Update Process

This architecture makes updates very **simple** and **intuitive**. We can update the Automation Manager Controller and the Automation Managers in **completely independent cycles** and at the pace we are comfortable with.

In order to update the Automation Manager Controller we will only have to trigger a new deployment using the CM Devops Center.

## Use Case - Update Process Automation Manager Controller

Let's see an example, we have an AMC installed for version **11.1.3**.

![AMC v 11.1.3](https://image.j-roque.com/posts/20250721-edge-deploy/amc1113.jpg)

Now we want to perform an update to version **11.1.5**.

![AMC v 11.1.5](https://image.j-roque.com/posts/20250721-edge-deploy/amc1115.jpg)

<video controls width="100%">
  <source src="https://image.j-roque.com/posts/20250721-edge-deploy/updateamc.mp4" type="video/mp4">
</video>

## Use Case - Update Process Automation Manager

For the Automation Manager we can undeploy, edit the manager version and trigger a new deploy. This will make the automation manager controller automatically deploy a new manager version.

![AM v 11.1.5](https://image.j-roque.com/posts/20250721-edge-deploy/am1115.jpg)

## Final Thoughts

We are now able to have most of the cloud benefits, also keeping our solution close to the shopfloor and staying easy to maintain and update.