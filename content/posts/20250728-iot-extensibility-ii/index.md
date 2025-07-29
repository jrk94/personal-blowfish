---
title: "Part II - Driver Extensibility"
description: "Driver Extensibility creating more dynamic driver definitions"
summary: "Driver Extensibility creating more dynamic driver definitions"
categories: ["Extensibility"]
tags: ["IoT", "Driver", "Tokens", "Templates"]
##externalUrl: ""
date: 2025-07-28
draft: false
authors:
  - Roque
---

Simple problems, have simple solutions, the trick is having simple solutions for very complex problems.

[Part I](https://j-roque.com/posts/20250728-iot-extensibility-i/)

## Overview

In part II, of the IoT Extensibility series, we will discuss mechanisms that add more dynamism in our driver definition. 

The driver definition is where the user maps the third party system interface into Connect IoT. It's where the user specifies what are the **events**, **commands** and **properties** he is interested in using and mapping.

We will see how the driver definition has different behaviors depending on the protocol it references. Some protocols are very strong typed, others require the user to map the machine specification into the MES.

## Templates

For protocols like, TCP-IP, REST or SECS/GEM, they are very dynamic and inform how the transport is created, but don't have a strict set of messages, they inform how the transport layer and structure of the message is created and delegate the message content format to the user, with some rules. These protocols force the user to have a very descriptive driver definition.

If I describe my **REST Client** to map a **POST** Request with a certain **payload** I will be forced to describe it in the driver definition. We can look back to our [3d Printer blog post](https://j-roque.com/posts/20250407-3dprinter/#material-handling) to see an example.

![Command Definition](https://image.j-roque.com/posts/20250407-3dPrinter/img/mes-startjob.png)

![Command Parameter Definition](https://image.j-roque.com/posts/20250407-3dPrinter/img/mes-startjobarguments.png)

There are other types of protocols where the full protocol is a set of strong typed more or less unchanging messages. For example, IPC-CFX is a strong typed protocol where all the messages and message format are known, or the File Raw protocol where all the events and commands are known.

These templates are provided in the driver level. When we [create a driver](https://developer.criticalmanufacturing.com/explore/guides/customizations/automation/customization-components/customization_driver/) using the CM CLI with [cmf new iot driver](https://criticalmanufacturing.github.io/cli/03-explore/commands/new_iot_driver/). The driver will be generated with a **templates** folder.

These are json files that describe, events, commands properties. Here the user can create predefined driver definitions, that will automatically be imported to the system.

Here we can see an example of the `File Raw` command `Exists` as defined as a template command.

```json
{
  "command": [
      {
          "Name": "Exists",
          "Description": "Checks if the path location exists.",
          "DeviceCommandId": "Exists",
          "ExtendedData": {
              "commandType": "Exists"
          },
          "CommandParameters": [
              {
                  "Name": "path",
                  "Description": "Path of the file",
                  "Order": 1,
                  "DataType": "String",
                  "AutomationProtocolDataType": "String",
                  "DefaultValue": "",
                  "IsMandatory": true,
                  "ExtendedData": {}
              },
              {
                  "Name": "attempts",
                  "Description": "Number of retry times",
                  "Order": 2,
                  "DataType": "Integer",
                  "AutomationProtocolDataType": "Integer",
                  "DefaultValue": 1,
                  "IsMandatory": true,
                  "ExtendedData": {}
              },
              {
                  "Name": "sleepBetweenAttempts",
                  "Description": "Waiting time between retries",
                  "Order": 3,
                  "DataType": "Integer",
                  "AutomationProtocolDataType": "Integer",
                  "DefaultValue": 1000,
                  "IsMandatory": true,
                  "ExtendedData": {}
              }
          ]
      }
  ]
}
```

We can now have a mix between driver definitions with user added fields and fields that are provided by the driver. We can also, have simply blank driver definitions if we just use the definition provided by the driver.

![Exists Example](https://image.j-roque.com/posts/20250728-iot-extensibility-ii/filerawexiststemplate.gif)

## Custom Templates

One interesting feature that is also supported is overriding the driver definition template via the controller.

This can be quite helpful in order to override or add templates provided by the driver. The user can add his own events and commands and update existing one's.

The user can then expand the existing driver definition template with his own. For example, for IPC-CFX there is an extensive library of commands and events and not all of them are declared out of the box by the driver.

The user can add them by using the `Custom Template` task, and specifying in a json format the changes he wishes to do to the template driver definition.

In this example, I am creating my own event based of the File Raw protocol.

![Custom Templates Example](https://image.j-roque.com/posts/20250728-iot-extensibility-ii/custom_templates_example.gif)

## Tokenization

A common pattern that we see in the need for dynamism when defining equipment interfaces is when there are a set of equipment that are virtually the same, but have a different identifier.

Connect IoT provides a simple mechanism to address this use case, by using tokens. The system allows the user to access a set of default tokens, and also information about the appended iot entity.

We can access and use the appended iot entity attributes and properties to allow us to construct our property device ids. Tokens are defined by being enclosed in ${<value>} or $(<value>).

## Tokenization - Use Case

A common use case is an OPC-UA server, that maps the tags of the machines with the same routing, only varies by adding the machine name in the routing node structure.

Let's image this use case, where we have `Station 01` and `Station 02`, the node for the production status is defined `ns=4;s=Station01.production.status` and `ns=4;s=Station02.production.status` respectively. Notice that both these node ids are very similar, they only change by adding the specific machine identifier in the node structure.

![Token Diagram](https://image.j-roque.com/posts/20250728-iot-extensibility-ii/tokendiagram.png)

We can leverage our IoT Architecture where we have the `Resource` as the appended iot entity and use the `Resource` Name as a key for our token replacement. Now we can reuse the same driver definition for all machines, by just replacing where there was the machine name by the corresponding token `${Name}`.

Using tokens can be a very powerful abstraction tool, that easily saves a lot of time and maintainability by saving the amount of driver definitions the user needs to maintain. A common drawback is when the user "abuses" the use of tokens. A token is by definition a dynamic resolution of a value, if you have several tokens or combination of tokens it can become very hard to troubleshoot if there is a problem in the final value resolution.

![Token Example](https://image.j-roque.com/posts/20250728-iot-extensibility-ii/token_example.png)

## Final Thoughts

We are continuing our journey through IoT Extensibility, in part II we focused on driver extensibility. How we could make the user life easier by supplying out of the box driver definitions, how we can later override and expand them and finally the use of tokens to make our driver definition much more dynamic. In part III, we will focus on customization, how we can leverage customization entrypoints to build very dynamic use cases.