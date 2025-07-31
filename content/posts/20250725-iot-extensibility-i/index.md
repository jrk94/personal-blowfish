---
title: "Part I - IoT Extensibility"
description: "IoT Extensibility, how we can leverage IoT extensibility to simplify complex scenarios"
summary: "IoT Extensibility, how we can leverage IoT extensibility to simplify complex scenarios"
categories: ["Extensibility"]
tags: ["IoT", "Standalone Workflows", "Architecture"]
##externalUrl: ""
date: 2025-07-24
draft: false
authors:
  - Roque
---

Simple problems, have simple solutions, the trick is having simple solutions for very complex problems.

## Overview

In this blog post we will go into some detail in understanding how we can leverage all that is built in into Critical Manufacturing Connect IoT platform in order to solve complex scenarios.

There are typically two axis of complexity, scalability and variability.

![Scalability vs Variability](https://image.j-roque.com/posts/20250725-iot-extensibility-i/scalability_vs_variability_chart.png)

`Scalability` can be defined in a scenario as a need to replicate or reuse with minimal effort and change. If for example, I have a hundred machines in my shopfloor with a similar process and of a similar type, a scalable solution is one that minimizes the effort of solving the same problem a hundred times. 

`Variability` is different dimension, how does your system model high change scenarios. One can think of a scenario where we have machines, that work partially in the same way but that require specific change. This is a common use case where we have a set of machines that have the same process, but are from different vendors or have a different model with different features.

`Complexity` grows as your variability and scalability grow and it grows exponentially.

## Architecture

It all starts with a flexible architecture. In a previous post [here](https://j-roque.com/posts/20250325-connectiotstructure/) we've already gone into detail about how the Connect IoT structure works and how we can leverage it.

In Critical Manufacturing Connect IoT we already have a **big split** between what are the **definition**'s of our integrations and what are the **instances** that are running. This allows us to leverage the same definitions across `N` instances.

![Multiple Instances](https://image.j-roque.com/posts/20250325-observability/img/multiplesameinstances.png)

This is very important, defining the **architecture** of your solution is often the **deciding factor in your project's success**.

### MES Entity

An instance is essentially the coming together of two sets of definitions. The integration definition given by the `Automation Controller` and an `MES Entity`. The entity that is appended to your controller is the premium hook your integration will have into the MES. It will serve to disambiguate between different controller instances and will allow you to provide context to your integration.

The simplest and most common use case is you have an MES Resource and have a controller instance that will control and interact with that Resource. If you have one hundred resources that work in the same way for the same process, you will have a hundred controller instances which leverage the same Automation Controller and will only change the MES Resource that they will hook themselves to.

Notice how powerful this is, we can have direct access to all the properties and attributes of the MES Entity that is attached to the instance, inside that same instance. We can also, have a different MES Entity per Automation Controller and per Driver. Allowing to even in the same controller have an MES entity responsible for the controller and different entities responsible for each driver.

Imagine the scenario where we have a machine with load ports, one parent with multiple child resources. The load ports can be modeled as MES Resources of type `Load Port` and the main Resource is a `Process` Resource.

![Load Port Scenario](https://image.j-roque.com/posts/20250725-iot-extensibility-i/loadports.png)

We can have the Controller hooked at the main resource and then each driver hooked at their matching load port. This way your integration will have the full context of the specific Resource they are controlling. This means the configuration and specific characteristics of the integration are not on the integration level, but on the Resource level. 

This is the proper level, as the **Resource** is the **domain/functional level**, the **integration** should only be interested on **interfacing** and the dynamically responding to functional changes.

![Load Port Instance](https://image.j-roque.com/posts/20250725-iot-extensibility-i/loadPortsMESRelations.png)

Now we can extrapolate to `N` instances.

![Load Port N Instances](https://image.j-roque.com/posts/20250725-iot-extensibility-i/loadPortsMESRelationsN.png)

---

Another pattern is also very common, which is an **integration level component** that **interacts with several resources**. 

We have a very common industry case, the OPC-UA server that acts as a line/area controller. In this use case we can also leverage the MES modeling to contextualize our integration. We can have an **MES Resource of type Component**, that is a **sub resource of all the resources it controls**. 

![Component Resource](https://image.j-roque.com/posts/20250725-iot-extensibility-i/componentresource.png)

With this relationship, we can have a dynamic discovery of all the resources that are controlled by the OPC-UA and create an integration that is dynamic and also that is more efficient. In this case, we won't need one OPC-UA Connection per Resource, but we will leverage one connection for all our Resources.

![Component Resource N](https://image.j-roque.com/posts/20250725-iot-extensibility-i/componentresourceinstance.png)

We can already see how the **MES modelling can be huge boon for our integration**. 

Architecture not only simplifies our life as integrators, but can have a multiplying effect on efficiency and simplicity.

## Standalone Workflows

Standalone workflows are very powerful abstraction points in the controller logic. They act as **touch-points that the user can reuse across multiple controllers**.

Let's imagine a scenario where we have a workflow that perform a sequence of tasks, task **A**, **B**, **C** and **D**. In our next integration we notice that in fact he has exactly the same use case as before but that instead of **A** and **B** he has only **E**.

![A B C D E](https://image.j-roque.com/posts/20250725-iot-extensibility-i/abcde.png)

We can extract the feature set that is the grouping of tasks C and D, into a standalone workflow. 

![Workflow A](https://image.j-roque.com/posts/20250725-iot-extensibility-i/workflowA.png)

This standalone workflow **can then be used across all your controllers**.

![Workflow A use](https://image.j-roque.com/posts/20250725-iot-extensibility-i/workflowAuse.png)

These workflows are ephemeral, similar to sub-workflows, they aren't meant to have long lived listeners. They also do not support driver actions as they are dependant on each controller's driver definition.

## Standalone Workflows - Use Case

Let's see a use case using the MES.

We will create a workflow that always saves a set of data in the persistency layer and broadcasts it to the Message Bus. This is a common scenario for abstraction, where you are able to **abstract either calls to the MES, interfaces to third party systems or data transformations**.

In our use case we have two controllers with completely different drivers, one is for file raw and the other is for tcp-ip.

![File Raw example](https://image.j-roque.com/posts/20250725-iot-extensibility-i/filerawexample.png)

![TCP-IP example](https://image.j-roque.com/posts/20250725-iot-extensibility-i/tcpipexample.png)

Both have completely different entrypoints and different end conditions but have similar subsets of logic. We can abstract this logic into a standalone workflow. 

Let's see how it would look. We can create our standalone workflow:

![Standalone Workflow example](https://image.j-roque.com/posts/20250725-iot-extensibility-i/standaloneworkflow.png)

In our standalone workflow we are able to define a set of inputs and outputs for our workflow using the start and end tasks. This is then what the user will be able to set from outside the workflow.

Now it is available and we can remake our controllers to leverage this shared workflow.

![File Raw Standalone Workflow example](https://image.j-roque.com/posts/20250725-iot-extensibility-i/fileraw_example_standalone.png)

![TCP-IP Standalone Workflow example](https://image.j-roque.com/posts/20250725-iot-extensibility-i/tcpip_example_standalone.png)

Even in such a simple integration it can heavily improve the **readability** and **simplicity** of the workflow.

All changes now performed to the standalone workflow will impact the controller that reference it. 

This can be a two way street, on one side this allows us to abstract and centralize this complexity and simplify our workflows, the flip side of this is that a change on this workflow can have impact on several controllers.

## Final Thoughts

We have started our journey in IoT Extensibility for part I we focused on leveraging the architecture and how we can use standalone workflows to simplify our controller logic. For part II we will focus on driver extensibility.