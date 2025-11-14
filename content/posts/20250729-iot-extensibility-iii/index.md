---
title: "Part III - Customization"
description: "Customization - making it complex, in order to make it simple"
summary: "Customization - making it complex, in order to make it simple"
categories: ["Extensibility"]
tags: ["IoT", "Customization"]
##externalUrl: ""
date: 2025-07-29
draft: false
authors:
  - Roque
---

Making it complex, in order to make it simple.

[Part I](https://j-roque.com/posts/20250725-iot-extensibility-i/)<br>
[Part II](https://j-roque.com/posts/20250728-iot-extensibility-ii/)

## Overview

In **part III**, of the **IoT Extensibility series**, is dedicated to customization.

These are very advanced use cases, but that allow for very generic solutions. 

The use of customization can be a big leverage to create abstracted solutions that fit your solution and can have an exponential impact in simplifying your integration process.

Customization is always a trade-off, in this post we will talk about about Connect IoT sandboxed customization, which mitigates some of the impact of customization as a choice, but nevertheless customization has an impact in maintainability.

All of these interactions depend on particular protocols. Protocols may support events, commands and property actions, or can just support a subset of them.

## Dynamically Perform Driver Actions

We have discussed, in previous blogs, the use of strategies to make your controller and driver definition more dynamic. These solutions are still very hooked to fixed structures, known before run time. 

There are use cases where we wish to have an even more dynamic approach, where what our driver should do depends on an external resolution.

Let's see an example. 

Imagine we have a **command that is executed when receiving a message bus message**, this is very simple to do with a message bus listener task and an equipment command task. But the equipment command task requires the user to know beforehand which command will be executed.

![Message Bus Command](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/mbmessagecommand.png)

What if we reach a very dynamic scenario where only the outside world knows which command will be executed? 

Where we want the message bus message to hold not just the input information of the command but the whole command definition.

In order to achieve this we can either use the `Driver Actions` tasks or the `Code Task` which has full access to the Driver API. We have a set of possible actions, executing commands, get and set of properties and create events.

---

## Use Case - Custom Command

In all the drivers that support custom extensibility, there is a subsection dedicated to the format of the message they are expecting and the topic.

For File Raw, we can [see](https://help.criticalmanufacturing.com/userguide/automation/definition/automation-protocol/automation-protocol-protocols/driver_fileraw/?h=driver+file+raw) that in order to perform commands it is expecting a command in the topic `connect.iot.driver.fileBased.executeCommand` and gives us the following example payload:

```json
{
    "command": {
        "name": "FileOrDirectoryExists",
        "deviceId": "Exists",
        "extendedData": { "commandType": "Exists" },
        "parameters": [
            { "name": "path", "dataType": "String", "deviceType": "String" },
        ],
    },
    "parameters": {
        "path": "c:/temp/file.csv",
}
```

With this strategy by using the `Send Notification to Driver` task we have fully delegated the driver command execution to the outside world. As long as it receives a payload that respects the contract of what is a command it is able to execute.

In this diagram we can see that on the left side, the command is being triggered by invoking the task `Execute Equipment Command` with the command `CreateDirectory`, which is known in the workflow statically. On the right side, we are using the task `Send Notification to Driver` and the workflow is fully agnostic to the command. In the second use case the external invoker can pass whichever definition he wishes, but has to provide the full command definition.

![Diagram Custom Command](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/diagramfilerawcustomcommand.png)

If we wanted to execute other types of commands our workflow wouldn't change as it would act as passthrough for the customization that is provided by the received message.

![Diagram Other Custom Command](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/diagramfilerawothercustomcommand.png)

Let's see it in the MES.

![Custom Command](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/filerawcustomcommand.png)

Note that in this case it is a message bus listener task, but we could also have for example a table resolution that would hold the metadata and would then invoke the command.

![Diagram Database Custom Command](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/diagramdatabase.png)

### Use Case - Code Task Send Raw

The same can be achieved with a `Code` Task. When dragging and dropping a task in the Connect IoT workflow, it will query the user if it will be used by a controller or by a driver. If the user selects driver, the driver api will be available in the code task.

![Code Task Send Raw Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/codetasksendrawworkflow.png)

![Code Task Send Raw](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/codetasksendraw.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {
    
    this.framework.driver.sendRaw("connect.iot.driver.fileBased.executeCommand", inputs.command);
}
```

Notice that by accessing the `this.framework.driver` variable we have the full driver api. In this example, we are similarly as above, passing on the full on command definition. 

### Use Case - Code Task Execute Command

Another possibility is when the command exists as part of the driver definition, then we can simplify the work of invoking the command.

We can leverage the driver definition and retrieve the command definition from the driver definition.

![Code Task Execute Command Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/codetaskexecutecommandworkflow.png)

![Code Task Execute Command](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/codetaskexecutecommand.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {
    
    // Search for our command in the driver definition
    const command = this.framework.driver.automationControllerDriverDefinition.AutomationDriverDefinition.Commands.find(tt => tt.Name == inputs.commandName);
    let paramMap = new Map<string, any>();

    if(command == null) {
        return {};
    }

    for(const item of command.Parameters) {
        // Set Command Parameters
        paramMap.set(item.Name, inputs.parameters[item.Name]);
    }

    await this.framework.driver.executeCommand(command, paramMap);
}
```

Now the responsibility is mixed and the external caller does not have to know the full command definition, he just needs to know the command name and the argument values for the command. We can leverage the driver definition contract already defined and just expose the invocation by name and arguments.

---

## Use Case - Custom Event

The events are split between two different actions that may happen sequentially or differed in time.

The **first action is registering the event**. This is responsible for notifying the driver that the event exists and it has a particular structure. 

A **second action is subscribing to the event**. Where the task notifies the driver that it wants to receive a particular event whenever it happens in the driver. Note that, an event listener may be created and destroyed throughout the workflow lifecycle and this may be an important element of your implementation.

This would be analogous to having an `Equipment Event` task, that has `Auto Enable` as **false** and only when a particular action occurs, will it toggle to true or false.

![Toggle Event Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/toggleeqevent.png)

### Use Case - Send Raw

Therefore, in order to provide the same behavior we require two different tasks, one will be `Send Notification to Driver` where we register our event and a `Subscribe in Driver` where we create our subscription. 

![Custom Event Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/customevent.png)

In this example we are registering and immediately subscribing, this is not mandatory, subscribing may happen any time after registering. 

The `Send Notification to Driver` task will receive as type, the specific type for the File Raw Protocol `connect.iot.driver.fileBased.registerEvent` and a payload with the event structure.

```json
{
    "event": [
        {
            "Name": "MyEvent",
            "Description": "Triggered when the watcher detects a file that was not previously identified appears",
            "DeviceEventId": "MyEvent",
            "IsEnabled": true,
            "ExtendedData": {
                "eventTrigger": "NewFile"
            },
            "EventProperties": [
                {
                    "Property": "FileName",
                    "Order": 1,
                    "ExtendedData": {}
                },
                (...)
            ]
        }
    ]
}
```

The matching `Subscribe in driver`, must receive a message type that is a concatenation of the subscribe topic with the event name `connect.iot.driver.fileBased.event.MyEvent`.

When there's an event occurrence, the task will output and the message output will contain the full event payload.

### Use Case - Code Task

A `Code` task cannot have live listeners. It is an ephemeral task with a defined time to live. As such, we cannot have event listeners in the code task. We can however handle all the parsing and construction of our events in order to handle the registration of the events.

![Code Custom Event Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/codetaskexecuteregistereventsworkflow.png)

![Code Custom Event](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/codetaskexecuteregisterevents.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {
    
    await this.framework.driver.notifyRaw("connect.iot.driver.fileBased.registerEvent", inputs.event);
}
```

Note that this approach may not seem immediately useful, but if we think we can frontload the event registration of all events and perform data transformation this becomes very useful. For example, with a simple loop we can register all the events.

![Code Custom Event Loop](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/codetaskexecuteregistereventsloop.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {
    
    for(const event of inputs.event) {
        await this.framework.driver.notifyRaw("connect.iot.driver.fileBased.registerEvent", event);
    }
}
```

---

## Use Case - Get Property

For protocols, like **OPC-UA** it is very common to want to retrieve a particular node id and not register a full on event. In these types of use cases it is very helpful to retrieve a specific property.

In this example we can see that **whenever we receive a message bus message, we will reply back with to the message bus with the result of the get property**.

![Get Property Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/getpropertyreplyworkflow.png)

### Use Case - Send Raw

The same can be achieved with the task `Send Notification to Driver`. In this task we will specify our type `connect.iot.driver.opcua.getPropertiesValues` and then our content payload. In the OPC-UA documentation we already see an [example](https://help.criticalmanufacturing.com/userguide/automation/automation-protocol/automation-protocol-protocols/driver_opcua/?h=opc#example).

The content that has to be provided is an object with the property name.
```json
[
  {
    "name": "status",
    "deviceId": "ns=2;s=Station01.production.status",
    "dataType": "String"
  }
]
```

The reply will be an object containing the information of the tag and the value retrieved.

```json
[
  {
    "propertyName": "status",
    "originalValue": {
      "dataType": "String",
      "value": "Productive",
    },
    "value": "Productive",
  }
]
```

![Get Property Send Notification Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/getpropertysendnotification.png)

As you can see this is very versatile **you can specify a full array of properties to retrieve**. Also, notice that we've used before the Send Notification to driver task, this task is quite generic what does the heavy lifting is the type and the content and all of this can be manipulated in the inputs.

### Use Case - Code Task

The Code Task can be very helpful for this use case. It allows us to perform the same action of the Send Notification to Driver task.

![Get Property Code Task Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/getpropertycodetask.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {
    outputs.reply.emit(await this.framework.driver.sendRaw("connect.iot.driver.opcua.getPropertiesValues", inputs.properties));
}
```

But more importantly, it simplifies the process of the data preparation. For example, we can already prepare the data by trimming all the information that is not needed. 

Let's imagine for this case, that my final object to reply to the outside world is just a name and value pair.

![Get Property Code Task Reply Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/getpropertiescodetaskreply.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {
    type Subset = {
        propertyName: string;
        value: any;
    };

    const values = await this.framework.driver.sendRaw("connect.iot.driver.opcua.getPropertiesValues", inputs.properties) as unknown as Subset[];
    outputs.reply.emit(values.map(value => ({ name: value.propertyName, value: value.value })));
}
```

### Use Case - Code Task Get Properties

Also, similarly to what we did for events we can also leverage the driver definition to retrieve the properties that are available and execute the get for them.

![Get Property Driver Definition Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/getpropertydriverdefinition.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {

    // Retrieve Property definition
    const propertyToGet = this.framework.driver.automationControllerDriverDefinition
        .AutomationDriverDefinition
        .Properties
        .find(propertyName => propertyName.Name === inputs.property);

    if (!propertyToGet) {
        throw Error(`No Property found`);
    }

    outputs.reply.emit(await this.framework.driver.getProperties([propertyToGet]));
}
```

With this approach the external caller can provide just a set of property names and will use all that is described in the driver definition.

---

## Use Case - Set Property

The set property is the mirror of the get property. The set property allows the user to write a particular value to a property.

![Set Property Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/setproperty.png)

### Use Case - Send Raw

In the documentation for **OPC-UA** we also have examples of how we can execute the set of properties. We will have to send a notification to type `connect.iot.driver.opcua.setPropertiesValues` with a payload:

```json
[
  {
    "property": {
      "name": "status",
      "deviceId": "ns=2;s=ns=2;s=Station01.production.status",
      "deviceType": "String"
    },
    "value": "Productive"
  }
]
```

Using the task **Send Notification to Driver** we can achieve this.

![Set Property Send Notification to Driver Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/sendnotificationsetprops.png)

### Use Case - Code Task

The **Send Notification to Driver** can also be achieved in the code task.

![Set Property Send Notification to Driver Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/setpropscode.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {

    await this.framework.driver.sendRaw("connect.iot.driver.opcua.setPropertiesValues", inputs.properties);
}
```

### Use Case - Code Task Set Properties

For the set properties we can also leverage the driver definition to retrieve our properties.

![Set Property Driver Definition Workflow](https://image.j-roque.com/posts/20250728-iot-extensibility-iii/setpropsdriverdefinition.png)

```ts
public async main(inputs: any, outputs: any): Promise<any> {
    const propertyToSet = new Map<AutomationProperty, any>();
    
    propertyToSet.set(this.framework.driver.automationControllerDriverDefinition
        .AutomationDriverDefinition
        .Properties
        .find(propertyName => propertyName.Name === inputs.property.Name), inputs.property.value);
    
    outputs.reply.emit(await this.framework.driver.setProperties(propertyToSet));
}
```

With this approach the external caller can provide just a set of property names and will use all that is described in the driver definition.

---

## Final Thoughts

Understanding these extension points is a big step in moving from a basic user capable of solving localized issues to being an advanced user capable of leveraging all that the system provides. 

These of course are tools that may be extremely helpful in some scenarios but that also may hinder a lot if used without care. Remember that every time you step into abstractions or generalizations you are moving away of your problem. 

Beware of the pitfall of the amazing abstractions that abstracts everything but doesn't solve or simplify the real life scenarios. Take care to not create magic solutions, as they have a habit of turning the magic against the wizard. Connect IoT brings a lot of transparency into the solution design, there is a trade off of losing transparency when you do these types of solutions.