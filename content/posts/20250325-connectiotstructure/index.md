---
title: "Overview Connect IoT Structure"
description: "Brief overview of the Connect IoT Model."
summary: "Brief overview of the Connect IoT Model."
categories: ["IoT"]
tags: []
#externalUrl: ""
date: 2025-03-25
draft: false
authors:
  - Roque
---

Brief overview of the Connect IoT structure.

## Connect IoT Structure

`Connect IoT` requires information about what protocols to be used. We will create one for `MQTT` and one for the `TCP-IP` driver. 

{{< alert "circle-info" >}}
**Info:** Automation Protocols are the meta definitions, all Automation Driver Definitions will inherit the definition of the protocol, even though the controller may override particular parameters. This is helpful for parameters that change dynamically. Normally, most parameters have default values, we define those in the Automation Protocol.
{{< /alert >}}

### Connect IoT Structure - Overview

Let's try and illustrate the advantages of this architecture with some diagrams.

An `Automation Protocol` can have multiple `Automation Driver Definitions`. They will all inherit the configurations defined in the `Automation Protocol`.

![Relation Protocol Driver Definition](https://image.j-roque.com/posts/20250325-observability/img/relationprotocoldriverdefinition.png)

The `Automation Controller` can then use several driver definitions and articulate them through low code to apply logic to the machine integrations.

![Relation Driver Definition Controller](https://image.j-roque.com/posts/20250325-observability/img/relationcontrollerdriverdefinition.png)

Finally, we have our entity that holds and runs our integration, that is the `Automation Manager` this entity will be able to run different controller instances. The instance is a link between an MES entity and an Automation Controller or Driver. This means that each iot instance can leverage an MES entity to imbue that integration with context.

### Running Structure

A manager can hold two (one controller and one driver) or `N` instances (one controller with `N` drivers or `N` controllers with `N` drivers). 

For example, let´s start with a simple use case, where a **manager has only one controller and two drivers**.

![Manager One Instance](https://image.j-roque.com/posts/20250325-observability/img/relationmanagercontrollerdrivers.png)

In this example, we have a controller instance with a controller `Automation Controller Wafer Station` that is appended to an entity `Resource` called `Wafer Station 01`. We have a driver instance with a driver definition `Automation Driver Definition MQTT Temperature / Humidity Sensor` append also to the same entity. Finally, we have an additional driver instance with a different driver definition `Automation Driver Definition TCP-IP Wafer Inspection` appended to a different entity type `Area`, with the name `Area Wafer Station`.

#### Running Structure - Replication

We can have the use case where we scale the number of instances, for example, per different resource (01 to `N`).

![Manager Multiple Same Instances](https://image.j-roque.com/posts/20250325-observability/img/multiplesameinstances.png)

In this scenario we have the same integration artifacts, but different appended entities. 

Here we see the full potential of having a separation between the concept of an `Automation Controller` and an `Automation Controller Instance`, we can reuse the same `Automation Controller`, creating `N` instances.

#### Running Structure - Mix

We can also have the scenario where we have multiple controllers in the same manager:

![Manager Multiple Different Instances](https://image.j-roque.com/posts/20250325-observability/img/multipledifferentinstances.png)

There is a lot of flexibility on how we can configure what is running in `Connect IoT`. We can also change this while the `Automation Manager` is running, and it will adapt. 

{{< alert "circle-info" >}}
**Info:** One thing that is always important to take into consideration is the resiliency level that we whish to have. A whole factory in the same `Automation Manager` means that an update or a catastrophic failure in an `Automation Manager` will impact everything in the shopfloor. It is common to find a middle road where the `Automation Manager` has a functional group of instances, for example, controlling all the machines in a line, if there is an issue, it will only impact a single line.
{{< /alert >}}

---

## Summary

Hope you enjoyed learning a bit about how you can turbocharge and leverage the Connect IoT model structure.

Thank you for reading !!!