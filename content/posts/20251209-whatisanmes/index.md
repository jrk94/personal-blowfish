---
title: "What is an MES?"
description: "What is an MES?"
summary: "What is an MES?"
categories: ["Basics"]
tags: ["MES"]
date: 2025-12-09
draft: false
authors:
  - Roque
---

We have dedicated a lot of effort to complex topics. Let's get back to the basics, this blog post focuses on explaining what is an MES and why your organization needs one.

## Overview

Every time we talk about a software that is intrinsicly B2B (business to business), we already lose a big part of the population and only talk to a subset of people. We also create knowledge silos and specific language that serve as barriers of entry for new people. Every person knows what an ecommerce website is, they interact with them everyday, but when people hear words like MES, PLM, ERP, for some is an intrinsic part of their day to day, for most others they are completely alien.

Before diving into explaining an MES it is important to understand that MES started very niche, but are becoming an ubiquitous presence in all things manufacturing, let's understand why.

An MES or `Manufacturing Execution System` is the software layer responsible for managing everything that happens in the shopfloor in order to be able to produce a finished good. It is the brain of a factory, it will coordinate people, machines, processes and quality to ensure a product is built correctly, efficiently and under total traceability and control.

## Digitalization Process

The MES has become the keystone of every digitalization process. It is the nexus of all other applications of the shopfloor. 

Throughout the years, it is common that small single target applications to target solving small problems end up being created. This may be great at solving localized problems, but as they grow either in scope, or in complexity, or as time goes by and the know-how is lost, this becomes an organizational wide problem.

The MES is commonly used as the tool to centralize all that happens in the shopfloor, bringing clarity and structure. It also either simplifies and deprecates those systems, introducing a standard way to address common issues. It also fosters collaboration between different departments, collaborating to find solutions to similar issues.

## System Stakeholders Shopfloor

Throughout the years as systems become more advanced a lot of barriers have become fuzzier. MES now does a lot more equipment integration than in the past, ERP systems now have MES modules and PLCs get smarter every year. But let's try and define what each of these words mean in order to understand what the MES typically is not.

Nevertheless, there is a typical a pyramid that illustrates how systems relate in the shopfloor.

![Manufacturing Software Pyramid](https://8sigma.eu/wp-content/uploads/2021/04/Untitled-design-12-640x360.png)

*Source: [8Sigma – Manufacturing Software Pyramid](https://8sigma.eu/erp-mes-scada-we-are-offering-an-all-in-one-solution-not/)*

Let's try and understand other key systems that are stakeholders of the shopfloor.

### ERP - Enterprise Resource Planning

An ERP system is a software that aggregates a lot of different facets of a company. It creates a central platform to manage finance, purchasing, inventory, HR, sales and planning. It tries to unify all of these diverse aspects into a cohesive and centralized system.

It tries to answer these questions: 

*What do we sell?* *What do we buy?* *What do we own?* *What do we owe?* *What should we produce next?*

In a shopfloor context it is `commonly known as the software application responsible for keeping track of money and orders`. It follows the money, by controlling inventory, stock levels, supplier lead times, raw material cost, payroll, etc. It is also responsible form managing purchase order requests, making sure they are delivered and understanding what was the profit and loss on that order.

The MES interacts with the ERP importing production orders and products, reporting material scraps, maintenances, etc. The MES is the boots on the ground for the ERP and is responsible for providing accurate and real data, moving from manual data.

### PLC - Programmable Logic Controller / SCADA - Supervisory Control and Data Acquisition

A PLC is an industrial computer. It's a computer that is able to live in the shopfloor and support all the dust, vibrations and so on. More importantly it is `focused on controlling a lot of inputs and outputs very fast`. It runs extensive logic to be run 24/7 for years controlling machines. It performs actions like motor on, motor off, rotate, read sensors, execute safety interlocks, detect alarms, etc. Over time the logic that they were able to run, became more complex as the computers themselves became more advanced.

A SCADA can be thought of as the eyes and ears of a PLC system. It displays machine status, shows alarms, logs historical data and allows operators to perform some machine interactions.

The MES interacts with those systems extracting information and controlling machines, via interfacing with PLCs and specific communication protocols.

### PLM - Product Lifecycle Management

A PLM is an engineering software focused on `managing a product's entire lifecycle`. It covers from idea → design → engineering → production → service → retirement. These are very specific systems focused on R&D (Research and Development) and in the productization process.

The MES can import information like ECADs, engineering drawings and product revision control. Creating an easy and detailed view from the production engineers into R&D specification.

## Learning by Analogy - Human Body

One of the most common analogies used to describe an MES is to think of the factory as a living and breathing human body.

![Human Body](https://cdn.britannica.com/07/192107-050-CE043374/anatomy-charts-human-body-muscle-systems-skeletal.jpg)

*Source: [human body](https://www.britannica.com/science/human-body)*

The `Manufacturing Execution System` is the factory’s *nervous system*. It controls three main systems:

### Nerves - Real-Time Awareness

It senses everything. It has `real-time awareness of everything that is happening`. Where the human body senses touch, pain, pressure, the MES senses, machine states (Productive, Scheduled Down, ...), operator actions, quality measurements. The `MES is collecting all the information that is happening in real-time and delivering in a way that is understandable`, just like the human body interacting with the environment.

### Reflexes - Makes Immediate Decisions

Your brain is `continuously managing your body`, either keeping the temperature stable, regulating hormones, killing cells with defects or simply breathing, all of this are reflex actions that we don't even think about but are key. Can you even imagine going through your day having to thing every few seconds to breath. This would consume so much mental power that you would be paralyzed, just trying to survive.

The MES is just like the human body, it automatically quarantines materials with defects, it rejects bad parts, provides machines and operators with recipe and job instructions and automatically routes all the materials you are producing in your shopfloor. 

Can you imagine managing a shopfloor where all of these actions are manual actions and accurately decide at every particular instance. How would you have time to think about improving? You would waste your time trying to keep breathing.

### Muscle Actions - Execution & Coordination

Your `nervous system is not just sensing and providing data, it also is coordinating complex systems in order to provide simple results`. 

Walking, an action that every human being is able to do after some months of existing is a highly complex action. 

Your leg and foot have to rise and fall a certain optimum level every time in sync and to top that the optimum level is always changing depending on the context (floor type, shoes, etc). What about the force your leg should do to place your foot down, this is also an constant optimization to keep you balanced and not exert wasteful force. This means that a simple action like walking when broken down in steps it is actually a set of highly dynamic complex actions.

The MES is exactly the same, a simple action like replenishing raw materials for the machines to be able to produce, becomes a complex inter-operation between systems. From interfacing with the machine to understand stock levels, to interacting with warehouse management softwares to create a request for material, to talk with Autonomous Mobile Robots (AMR), to perform the pick up and the machine replenishment. This can even become more complex if you wish to have algorithms to use first what you have available in the shopfloor, before requesting from the warehouse. 

`The MES is responsible to interface between all these different realities to transform a simple action like replenishment actually simple, when in fact is a set of dynamic and complex actions.`

### Other Parts

#### Organs & Muscles

We can think of the actual organs and muscles of the human body are all the machines, AMRs and operators that perform guided actions throughout the shop floor.

#### Blood

The blood of a shopfloor is no doubt all the materials, raw materials and spare parts, that continuously circulate in the factory. They move between different organs and muscles (machines and operators), in order to at the end create finished goods.

#### Brain (Long Term Planning)

No doubt the MES shares in the brain analogy with the ERP (Enterprise Resource Planning). While the MES is focused on short and medium term actions, the ERP is focused on creating and managing needs. It's like our long terms goals, of buying a new house or a new car. It is a software responsible for feeding the MES with orders the MES must fullfil, like build 10000 units of materials X. The MES will then translate that order into actionable actions, like the order will be fulfilled by machine A with the operator B, using the raw materials C.

The ERP will then be notified by the MES of how the order is going, mainly interested in things that affect costing and schedule of the order, like how much scrap is being generated and the macro stage of where the material is at and finally how many materials of that order have been finished.

#### Spinal Cord (Direct Machine Control)

PLC (Programmable Logic Controller) and Scada (Supervisory Control and Data Acquisition) systems serve as the middle man between the MES and the machine. They are responsible for direct machine control and safety. Imagine a conveyor belt and a set of machines. The MES is focused on understanding if the material is supposed to be at a particular machine, if the recipe of the machine is correct, if there are enough raw materials. The PLC is going to concern itself in moving the conveyor belt, in controlling the specific action the machine is doing in order to fulfill the recipe and making sure everything is done is safe manner.

ERP is the brain that plans the future, PLCs are the muscles that move, and MES is the nervous system that senses, decides, coordinates, and remembers everything happening right now.

## Final Thoughts

With a cheaper cost of entry and more intelligent machines, MES systems are becoming ever more prevalent. 

Controlling a shopfloor without an MES system is like trying to catch water with your hands, you will waste so much time trying to hold it, you won't have time to drink it. 

The MES is a key system for a digitalization process as it will democratize and universalize shopfloor control systems.

The age of AI also opens the opportunity for a more intelligent MES. The thinking factory blends the control and information gathering that MES lives and breathes by, with the potential for inference and knowledge reasoning that genAI brings to the table. 

{{< youtube M18HIc06qPU >}}