---
title: "Edge Observability"
description: "Bringing visibility to on Edge Deployments."
summary: "Bringing visibility to on Edge Deployments."
categories: ["IoT"]
tags: [ "MQTT", "TCP-IP", "DataPlatform"]
#externalUrl: ""
date: 2025-03-25
draft: false
authors:
  - Roque
---

A small use case on using `MQTT` and `TCP-IP` drivers for material tracking and resource state management.

# Overview

Previous use cases were very light on control, the greatest advantage of an MES system is not just the ability to collect information and provide context or generate reports, but more importantly to have **actionable control**.

Therefore, for this use case I chose to use a very simple, but common scenario where we have full automation for material tracking and state management. The twist, and it's a very common twist, is that the control is done when we bring two different data producing interfaces and use the MES to provide context and control.

In this example, `MQTT` will provide humidity and temperature control, if the values are above a particular threshold, it will send the machine into a SEMI-E10 `Unscheduled Down`.

Meanwhile, the `TCP-IP` interface is only concerned in controlling the material tracking through the machine. But what happens if we get an event saying that the material is in process, while the machine is an due to an invalid temperature or humidity `Unscheduled Down`?

We are going to find out!!!

We will then use this opportunity to backtrace what is happening through the native logging support and then to our subscription based **observability platform**.

---

## MES Model Overview

For this scenario we will have a resource `Wafer Preparation Station Cleanroom` which is handling materials `Wafers`:

**Resource View**:

When interacting with our `Resource` this is one of the most important views. In the left tab we can see that we are currently in the `Dispatch List` and we have `10 Materials` (10 wafers) that are dispatchable to this `Resource`. 

There is also additional information that refers to other actions and views on the Resource, like if the resource has Load Ports, Sub-Resources, Maintenances, and others which will not be relevant for our use case.

![Resource View](https://j-roque.com/posts/20250325-observability/img/resourceview.png)

**Area View**:

Notice that for this area, we only have one step, but we could have `N` steps and for our single step we have only one resource, we could have `N` resources. Also, right away we see that we have `10 Materials` (10 wafers) in our Step `Wafer Preparation` but currently those materials are not dispatched to our resource.

![Area View](https://j-roque.com/posts/20250325-observability/img/areaview.png)

**Facility View**:

![Facility View](https://j-roque.com/posts/20250325-observability/img/facilityview.png)

**Factory Explorer**: 

![Factory Explorer](https://j-roque.com/posts/20250325-observability/img/factoryexplorer.png)

There were other configurations that are relevant, like creating a product, defining a material flow, defining what services the resource provides. Explanations on these topics is out of scope for our goal today.

---

## Connect IoT Structure

`Connect IoT` requires information about what protocols to be used. We will create one for `MQTT` and one for the `TCP-IP` driver. 

{{< alert "circle-info" >}}
**Info:** Automation Protocols are the meta definitions, all Automation Driver Definitions will inherit the definition of the protocol, even though the controller may override particular parameters. This is helpful for parameters that change dynamically. Normally, most parameters have default values, we define those in the Automation Protocol.
{{< /alert >}}

### Connect IoT Structure - Overview

Let's try and illustrate the advantages of this architecture with some diagrams.

An `Automation Protocol` can have multiple `Automation Driver Definitions`. They will all inherit the configurations defined in the `Automation Protocol`.

![Relation Protocol Driver Definition](https://j-roque.com/posts/20250325-observability/img/relationprotocoldriverdefinition.png)

The `Automation Controller` can then use several driver definitions and articulate them through low code to apply logic to the machine integrations.

![Relation Driver Definition Controller](https://j-roque.com/posts/20250325-observability/img/relationcontrollerdriverdefinition.png)

Finally, we have our entity that holds and runs our integration, that is the `Automation Manager` this entity will be able to run different controller instances. The instance is a link between an MES entity and an Automation Controller or Driver. This means that each iot instance can leverage an MES entity to imbue that integration with context.

### Running Structure

A manager can hold two (one controller and one driver) or `N` instances (one controller with `N` drivers or `N` controllers with `N` drivers). 

For example, let´s start with a simple use case, where a **manager has only one controller and two drivers**, which will be the use case in this blog post.

![Manager One Instance](https://j-roque.com/posts/20250325-observability/img/relationmanagercontrollerdrivers.png)

In this example, we have a controller instance with a controller `Automation Controller Wafer Station` that is appended to an entity `Resource` called `Wafer Station 01`. We have a driver instance with a driver definition `Automation Driver Definition MQTT Temperature / Humidity Sensor` append also to the same entity. Finally, we have an additional driver instance with a different driver definition `Automation Driver Definition TCP-IP Wafer Inspection` appended to a different entity type `Area`, with the name `Area Wafer Station`.

#### Running Structure - Replication

We can have the use case where we scale the number of instances, for example, per different resource (01 to `N`).

![Manager Multiple Same Instances](https://j-roque.com/posts/20250325-observability/img/multiplesameinstances.png)

In this scenario we have the same integration artifacts, but different appended entities. 

Here we see the full potential of having a separation between the concept of an `Automation Controller` and an `Automation Controller Instance`, we can reuse the same `Automation Controller`, creating `N` instances.

#### Running Structure - Mix

We can also have the scenario where we have multiple controllers in the same manager:

![Manager Multiple Different Instances](https://j-roque.com/posts/20250325-observability/img/multipledifferentinstances.png)

There is a lot of flexibility on how we can configure what is running in `Connect IoT`. We can also change this while the `Automation Manager` is running, and it will adapt. 

{{< alert "circle-info" >}}
**Info:** One thing that is always important to take into consideration is the resiliency level that we whish to have. A whole factory in the same `Automation Manager` means that an update or a catastrophic failure in an `Automation Manager` will impact everything in the shopfloor. It is common to find a middle road where the `Automation Manager` has a functional group of instances, for example, controlling all the machines in a line, if there is an issue, it will only impact a single line.
{{< /alert >}}

---

### Create an Automation Protocol MQTT and TCP-IP

For now, we will create them with the default settings.

`MQTT` **Automation Protocol**:

![`MQTT` Automation Protocol](https://j-roque.com/posts/20250325-observability/img/mqtt_protocol.png)

`TCP-IP` **Automation Protocol**:

![TCP-IP Protocol](https://j-roque.com/posts/20250325-observability/img/tcp_protocol.png)

Notice that their settings are completely different. This is normal as each transport protocol has its own specificities. In `Connect IoT`, when creating a driver, you can specify all the settings that are particular to your driver.

---

### Create an Automation Driver Definition MQTT

The `Automation Driver Definition` will be where we map the relevant fields of the specification. Here is where we will configure all the events that we want to subscribe to or commands that we want to execute.

The tutorial has a very simple example of collecting temperatures and humidity readings. In this example, all topics have two levels i.e *waferprep/temperature/* or *waferprep/humidity/*, the second level *temperature* or *humidity* is what will inform the system if this value is for temperature or humidity.

{{< alert "circle-info" >}}
**Info:** In a production setting it is very common that the mapping of the system does not have a direct correlation with the MES. Here is where `Connect IoT` can serve as middleware to map everything into a common standard.
{{< /alert >}}

The `MQTT` driver in `Connect IoT` supports wildcards:
- '#' - can be used as a wildcard for all remaining levels of hierarchy. This means that it must be the final character in a subscription. For example: sensors/#
- '+' - can be used as a wildcard for a single level of hierarchy. For example: sensors/+/temperature/+

I will set the `Topic Name` as *waferprep/temperature/#* for temperature and *waferprep/humidity/#* for humidity.

In our example, we have to create two properties and an event to surface both properties the `Ambient Monitoring` event:

![MQTT Automation Driver Definition Property](https://j-roque.com/posts/20250325-observability/img/mqtt_driverdefinition.png)

Setting the system as such will mean that whenever there is a new value for temperature or humidity the event will be triggered.

---

### Create an Automation Driver Definition TCP-IP

The TCP-IP driver allows for active or passive communication. In other words it can behave as a server or a client. The TCP-IP driver is a simple driver as it does not implement a standard communication interface, but it is in fact a transport protocol. It is nevertheless very useful for integrating with simple machines. Machines like barcode readers, metrology or testing is not uncommon to have simple tcp-ip interfaces.

{{< alert "circle-info" >}}
**Info:** When facing a protocol that communicates over tcp-ip but has a complex set of handshakes or lifecycle, it is often simpler to create a new custom driver. For example, secs-gem can be over tcp-ip, or the Fuji-Nexim, or for example drivers that use XML over tcp-ip it's often easier to build a driver that manager all the communication interface. Instead of making a very complex controller logic on top of the tcp-ip driver.
{{< /alert >}}

#### Machine Lifecycle

Our TCP-IP machine will host a TCP-IP Server and will send two messages. 

- **MaterialIn** - Signals that a material has started being processed
- **MaterialOut** - Signals that a material has finished processing

Both these machines wait for a reply with acknowledgment, if there is no acknowledgement it will not proceed.

![TCP-IP Machine Flow](https://j-roque.com/posts/20250325-observability/img/tcp_machineflow.png)

#### Message Format

The machine messages will follow a defined format:

**<STX\><EventId\>,Product:<Product\>,Quantity:<Quantity\>,Material:<Material\><ETX\>**

In order to build a driver definition for the TCP-IP driver, we will have to understand the message and translate the message into an event.

![TCP-IP Message Deconstruct](https://j-roque.com/posts/20250325-observability/img/messagedeconstruct.png)

Let's now define what makes each member unique:
- **EventId** - Beginning of the message and a ','
- **Product** - Between a token Product: and a ','
- **Quantity** - Between a token Quantity: and a ','
- **Material** - Between a token Material: and an end of message

The message acknowledge command is similar:

**<STX\><EventId\>,<Material\><ETX\>**

#### Mapping to a Driver Definition - Event

Let's define our properties, we will use regular expressions to extract from the stream all the values that are relevant for us. 

- **EventId** - Beginning of the message and a ','

![Driver Definition Event Id](https://j-roque.com/posts/20250325-observability/img/driverdefinition_eventId.png)

- **Product** - Between a token Product: and a ',' (others are similar)

![Driver Definition Product](https://j-roque.com/posts/20250325-observability/img/driverdefinition_product.png)

- **Material** - Between a token Material: and an end of message

![Driver Definition Material](https://j-roque.com/posts/20250325-observability/img/driverdefinition_material.png)

And finally, the event will tie all of this together:

![Driver Definition Event](https://j-roque.com/posts/20250325-observability/img/driverdefinition_event.png)

#### Mapping to a Driver Definition - Command

We will create two commands, one for the **material in** and the other for the **material out**. With the flag `Command device id Usage` as `AtBeginning`, the system will automatically add the device id to our command.

![Driver Definition Commands](https://j-roque.com/posts/20250325-observability/img/driverdefinition_commands.png)

Let's now add our material parameter:

![Driver Definition Command Parameter](https://j-roque.com/posts/20250325-observability/img/driverdefinition_command_params.png)

---

### Create an Automation Controller

Creating our controller, we will specify that it has our two drivers, one for `MQTT` and one for `TCP-IP`.

![Automation Controller](https://j-roque.com/posts/20250325-observability/img/controller.png)

### Setup

In our setup page the template will automatically generate the two driver quickstart.

![Automation Controller Setup](https://j-roque.com/posts/20250325-observability/img/controller-setup.png)

{{< alert "circle-info" >}}
**Info:** In this example, I am statically defining address and port and other settings for convenience. In a normal productive use case these would be dynamically resolved, from an entity attribute, a table or some other way.
{{< /alert >}}

### Material Handling

The machine will provide two events, one that will signal the **material in** and the other will signal the **material out**. 

For this example, we have the same event handler for both events, this is common for tcp-ip as it's a transport protocol, for richer protocols this is less common.

![TCP-IP Material Handling](https://j-roque.com/posts/20250325-observability/img/tcp_event.png)

The workflow will be triggered by a TCP-IP message, with a break line. The workflow will then check the `eventId` against the `MaterialIn` or `MaterialOut`.

If the eventId matches the `MaterialIn`, we will retrieve the resource instance and call the `MES` service `TrackInMaterial`.

If the eventId matches the `MaterialOut`, we will simply call the `MES` service `TrackOutMaterial`.

If there is an unexpected eventId we will send an exception mentioning this is an unexpected eventId.

### Control Temperature and Humidity

We will use an MQTT integration to receive temperature and humidity values. If the values are above a particular threshold, we will change the current resource state to a state that matches an invalid state, the SEMI-E10 `Unscheduled Down`.

![MQTT Control Temperature and Humidity](https://j-roque.com/posts/20250325-observability/img/mqtt_event.png)

We will receive an event with temperature and humidity. If there´s no value yet defined for one the fields, they would be `undefined` therefore we normalize them to 0.

In the controller we will have two relevant conditions. If the temperature is **above 27ºC** or if the humidity is **above 50%** it will change the Resource state to `Unscheduled Down`.

---

## Running the Scenario

Let's see what we can do with this controller!!!

### Automation Manager

I connected the Controller to an Automation Manager, I am using `Hercules` to emulate TCP-IP Server and `mosquitto` to serve as an mqtt broker and to publish messages.

Let's start with the material tracking scenario. First, let's dispatch one of our wafer materials to our station.

![Dispatch Material](https://j-roque.com/posts/20250325-observability/img/dispatch-material.gif)

The material was at the dispatch list and was dispatched to the `Wafer Preparation Station`. When the material is allocated to the resource the actions possible to perform to the material are adapted to this new state.

{{< alert "circle-info" >}}
**Info:** In our use case, the allocation of material to a resource is one to one, but if you had `N` number of resources that could provide the same service to the material, you could dispatch to any one of them. For example, if we had a cluster of preparation stations, the material could be dispatched to any one of those stations.
{{< /alert >}}

### Material Lifecycle

Using the `TCP-IP` integration let´s now perform the material production cycle.

The `TCP-IP` simulator will send a Material In event:

**<STX\>MaterialIn,Product:DemoProduct,Quantity:1,Material:Wafer-01<ETX\>**

![Material In](https://j-roque.com/posts/20250325-observability/img/material%20tracking.gif)

The `TCP-IP` simulator will send a Material Out event:

![Material Out](https://j-roque.com/posts/20250325-observability/img/material-tracking-out.gif)

### Material Lifecycle - Logging Files

When we saw this scenario, we saw it through the lens of a console application logging. The log transports for Connect IoT are defined in the Automation Manager Configurations. 

There are three major types of logging transports

- **Console** - for development purposes
- **File** - normally used to persist logs to an external network drive 
- **OTLP** - which is the open telemetry transport.

![Manager View](https://j-roque.com/posts/20250325-observability/img/manager_view.png)

![Manager Configuration](https://j-roque.com/posts/20250325-observability/img/manager_configuration.png)

Let's take a look at the default file transport logging configurations:

```json
{
	"id": "Controllers",
	"type": "File",
	"options": {
		"filename": "${applicationName}_${date}.log",
		"dirname": "${temp}/ConnectIoT/Logs/Instances/${entityNameNormalized}/${componentId}",
		"level": "info",
		"timestampFormat": "HH:mm:ss.SSSSS",
		"maxSize": "10m",
		"maxFiles": "30d",
		"isEnabled": true
	},
	"applications": [
		"AutomationController"
	]
},
{
	"id": "Drivers",
	"type": "File",
	"options": {
		"filename": "${applicationName}_${date}.log",
		"dirname": "${temp}/ConnectIoT/Logs/Instances/${entityNameNormalized}/${componentId}",
		"level": "debug",
		"timestampFormat": "HH:mm:ss.SSSSS",
		"maxSize": "10m",
		"maxFiles": "30d",
		"isEnabled": true
	},
	"applications": [
		"Driver*"
	]
},
{
	"id": "ManagerAndMonitor",
	"type": "File",
	"options": {
		"filename": "${applicationName}_${date}.log",
		"dirname": "${temp}/Demo/ConnectIoT/Logs/${applicationName}/${managerId}",
		"level": "info",
		"timestampFormat": "HH:mm:ss.SSSSS",
		"maxSize": "10m",
		"maxFiles": "30d",
		"isEnabled": true
	},
	"applications": [
		"AutomationMonitor",
		"AutomationManager"
	]
}
```

The default configuration splits the file logging into three different transports. These will generate a folder tree structure. 

One transport will log all the information related to the **Controller**, another will have all the information regarding **Drivers** and the last one refers to the **manager and monitor** boot up. 

This configurations allow us to customize what is of interest to us to log, for example the level of verbosity and the roll over of the logs. By default, logs will be kept for 30 days and a new file is generated every 10 megabytes, all logs will have an entry with a timestamp. 

Let's take a look at the same scenario through the log files.

![Controller Directory Logs](https://j-roque.com/posts/20250325-observability/img/controller_file_dir.png)

The Logging structure is created using the dirname and filename specified in the configuration.

![Controller Logs](https://j-roque.com/posts/20250325-observability/img/controller_file.png)

One important highlight is that you can already see when the actions occurred in the Controller layer, which tasks activated in each page and each execution will have a unique identifier. The user can then leverage all this information to backtrace all information.

The logs are not as complete as the ones we saw in the console log, as the console log merges all logging so let´s take a look at the driver tcp-ip logs.

![Driver  Directory Logs](https://j-roque.com/posts/20250325-observability/img/driver_file_dir.png)

![Driver Logs](https://j-roque.com/posts/20250325-observability/img/driver_file.png)

We can see how helpful it can be for troubleshooting to have this segmentation by component. In the **driver** the user can expect to see all the i**nformation regarding the communication layer**, for example are all the ports open, are the events and commands registered. Whereas, in the **controller** we see **information related to our integration and business logic**.

### State Change

Another scenario is when there is a temperature or humidity above a particular threshold. For this one we will use the `MQTT` protocol.

![State Change](https://j-roque.com/posts/20250325-observability/img/state-change.gif)

When a machine has an invalid state to produce like `Unscheduled Down`, the `MES` prevents work from being done to the material.

If we try to track in or out using our tcp-ip integration the `MES` will give an error and not send an acknowledgment message.

![TCP-IP Fail](https://j-roque.com/posts/20250325-observability/img/tcpip_fail.gif)

### State Change - Logging Files

If we investigate our logging files, we can see that our driver has no errors.

![Driver TCP-IP](https://j-roque.com/posts/20250325-observability/img/driver_file_tcp.png)

In the Controller, where we have our business logic, we can now see in detail what has occurred.

![Controller TCP-IP Fail](https://j-roque.com/posts/20250325-observability/img/controller_file_fail.png)

We have our mqtt driver changing temperatures and we have our MES request failure, showcasing that the material cannot be tracked in as the Resource is in an invalid state.

---

## Observability

One of the issues of on edge devices is centralized visibility on everything that happens. This is may not be major problem (it still is 😉) if all your devices run in the same kubernetes cluster and everything is one click away or one log tool away, but for automation that is not enough. Reading file logs is also a very cumbersome and static post factum analysis.

This is why we launched our observability platform.

### Observability - Automation

Let's see the difference.

![Observability Automation](https://j-roque.com/posts/20250325-observability/img/observability_IoT_GUI.gif)

We are now seeing remotely everything that is happening on our Connect IoT application. Not only that, but we are able to provide filters from Environment, to Manager, to Component. This way we can pinpoint what part of our components is causing us issues. We can also perform custom queries to filter the data. All of this is dynamic and in near realtime. We can also perform macro analysis of a system by understanding our ration of how many messages have what verbosity and what are the ration between them. In my use case I had already some error and warning messages. We can also adjust the time window as we wish, for this scenario we were interested in the latest messages, but we could filter for a particular timestamp where we know something unexpected happened in the shopfloor.

### Observability - BackEnd

Of course, observability is not only about automation. All components of the MES are represented in a dashboard, some of them even have specific dashboards.

Let's backtrace the actions we performed from the perspective of our backend application. One of the services we used was the `TrackInMaterial`.

![Observability Backend Services](https://j-roque.com/posts/20250325-observability/img/observability_BE.gif)

We are able to have multiple Environments being tracked at the same time. We can then filter by component, in our use case we were interested in the backend execution of the service, so we filtered by `host` and then search for `TrackInMaterial`. After finding our service call, we can then see the logs just for that transaction and even see a trace view. We can see everything our service is executing and how much time we are spending in each action.

It is also very helpful to bring visibility to errors. Let's see what the error scenario of trying to perform track in of a material to a Resource looks like.

![Observability Error Backend Services](https://j-roque.com/posts/20250325-observability/img/observability_Error.gif)

We also have aggregate dashboards where we can see which are the most requested endpoints and the ones that take the most time.

![Observability Backend Services Aggregated](https://j-roque.com/posts/20250325-observability/img/observability_BE_Metrics.gif)

![Observability Backend Trace Services Aggregated](https://j-roque.com/posts/20250325-observability/img/observability_BE_Trace_Metrics.gif)

### Observability - FrontEnd

One of the interesting dashboards we also have is the one that tracks user interactions with the user interface.

![Observability UI](https://j-roque.com/posts/20250325-observability/img/observability_UI.gif)

Throughout our use case the UI we traversed more was the view from the Resource. We can see that the material page is the Ui that took more time. This is very important to understand the health of the system and to understand if user perceived unresponsiveness comes from the frontend, backend or if it's just a perception and to troubleshoot what are the pages that could be optimized.

We can also see that I accessed the MES both from a 1080p monitor and 2k monitor. This is interesting information to understand what are the type of users for our dashboards. If for example, most of the users use mobile or use a specific screen size, the UIs can be optimized to be more responsive to those screen sizes.

## Summary

There are many more dashboards and views in our Observability module, but I think it is easy to see the immense benefit of having a centralized platform where you can see everything that is happening in your several MES environments.

Thank you for reading !!!