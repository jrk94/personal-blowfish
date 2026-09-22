---
title: "Integrating with OIB: Board Gatekeeper and Eventing"
description: "What ASMPT's Operations Information Broker actually is, and why the SMT line asks the MES for permission."
summary: "What ASMPT's Operations Information Broker actually is, and why the SMT line asks the MES for permission."
categories: ["IoT"]
tags: ["OIB", "Electronics", "SMT", "ConnectIoT", "WCF"]
##externalUrl: ""
date: 2026-09-08
draft: false
authors:
  - Roque
---

Right now, on some SMT line, a printed circuit board is sitting motionless in a conveyor while the line waits — because a web service has not yet said whether that board may continue.

That is *ASMPT WORKS* OIB working exactly as designed.

## Overview

The **Operations Information Broker** — OIB — is ASMPT's integration platform for the WORKS software suite. It has been around since 2008, it is built on WCF web services, and it is the only sanctioned way into SIPLACE Pro, WORKS Setup Center, Line Control, Traceability, and the rest of the ASMPT stack. The native interfaces those products used to expose (COM, direct SQL, Secs-GeM, parsed XML files) are gone from public use.

These are the main five things OIB lets your software do:

- query for operational data
- create or modify data
- control the line
- be informed about line events
- **be requested by the line for individual decisions**

Four of those are ordinary. The fifth is the one this post is about.

This post covers what OIB is and how CM has learned from it and has created turn key integration strategies.

---

## What OIB Actually Is

OIB is not an MES and does not try to be one. It is a **Service Oriented Architecture** layer whose only job is to unify and abstract the shop floor so that higher-level software stops caring about deployment topology. Its goal is to monitor, control and handle the logistics of an SMT line that runs ASMPT machines.

![ASMPT Line](https://image.j-roque.com/posts/20260908-oib/asmpt_asm_line.png)

It splits into two halves.

**OIB Core** is the infrastructure: the OIB database plus a set of core services.

| Core service | Responsibility |
|:--|:--|
| `Configuration Manager` | Owns and persists the OIB Factory Layout |
| `Service Locator` | Service registration and discovery — the yellow pages |
| `WS Eventing` | Guaranteed-delivery event dispatch |
| `Display Service` | Messages and questions to line-side and station viewers |
| `Central Settings` | Shared configuration, scoped per factory element |
| `Factory Calendar` | Shifts, maintenance downtime, appointments |
| `User Manager` | Authentication and authorisation — local users, AD, or both |
| `Health Check` | Polls registered services for availability; can e-mail admins |

Each of those has an ASM Studio plugin in front of it — Factory Explorer, Service Manager, Factory Calendar, User Manager, Operation Manager, and a Test Box for poking endpoints by hand. Configuration is a GUI activity.

**OIB Adapters** are the endpoints that expose actual business functionality:

| Adapter | What it gives you |
|:--|:--|
| `SPI` (SIPLACE Pro Interface) | CRUD on SIPLACE Pro — components and shapes, placement lists, panels and sub-panels, setups, recipes, line and machine configuration |
| `Optimizer` | Configure, start/stop, and read results of job optimisation; progress events |
| `Line Control` | Start/continue/stop the line, block/unblock station conveyors, download recipes, read line and recipe status |
| `Setup Center` | Read/write packaging units; events for feeder, table and material movement and quantity change |
| `Traceability` | Per-board production result — where, when, which errors, which recipe and order, which packaging units |
| `Monitoring` (OIS) | Real-time machine state, pickup and reject rate, active recipe, feeder track info |
| `Board Gatekeeper` | Board-level GO/NOGO at a scan point |
| `Material Manager` | Material flow events, including moisture sensitivity |
| `Maintenance Data Interface` | Station and setup configuration with unique IDs, plus station health data every 15 minutes |

Core is typically installed once per factory on a central server, while an Adapter instance is installed for each installed instance of the software product it fronts — not a fixed rule of "one per line". For Setup Center that instance happens to be per-line: a customer with ten lines running Setup Center has ten Setup Center Adapter instances, each registering independently. A different adapter could just as easily be one per factory or one per station, depending on how that product is deployed.

In practice the split lands like this: the central server carries OIB Core, the OIB DB, Eventing, and the SPI Adapter; each line computer carries its own Setup Center Adapter, Line Control Adapter, Monitoring Adapter, Traceability, and Board Gatekeeper.

### The Factory Layout

Part of the OIB concept is a persisted model of the factory, following ISA-95:

```text
Enterprise
└── Site
    └── Area              (optional — lines can attach directly to sites)
        └── Production line
            └── Work cell (can nest)
```

A production line is a printer, some placement stations, an oven. A work cell is one of those stations. Areas are optional and work cells nest, which is the escape hatch for real factories that never match the diagram.

![ASM Works](https://image.j-roque.com/posts/20260908-oib/asm_works.png)

This layout lives in the OIB DB, is edited in WORKS Studio under Factory Explorer → Factory Layout, and can be read and written programmatically through the Configuration Manager. It is not decoration — discovery is built on top of it, and so are Central Settings and the Factory Calendar, both of which attach to individual factory elements.

---

## Board Gatekeeper

Process control on an ASMPT line belongs to SIPLACE Pro Line Control. **WORKS Board Gatekeeper** — BGK — extends it with something Line Control does not do: it lets an external system make a decision about a specific PCB, at a specific point in the line, *before* that board is allowed to continue.

Two capabilities:

- get informed about PCBs — optionally identified by barcode — transported anywhere in the factory line
- stop a PCB in the conveyor if requirements are not fulfilled

BGK is an agent sitting between barcode scanners and conveyor hardware on one side, and your MES on the other. You choose where the scan points are — line entry, line exit, into a station, out of a station.

![ASM BGK](https://image.j-roque.com/posts/20260908-oib/asm_bgk.png)

It runs in one of three modes:

| Mode | Conveyor stop | MES informed | Use case |
|:--|:--|:--|:--|
| `Notification` | No | Yes | Track boards, don't gate them |
| `Interlocking` | Yes | Yes, and waits for confirmation | Real gatekeeping |
| `No MES` | No | No | Barcode whispering only |

`Notification` is tracking — you learn that a registered board entered a machine. 
`Interlocking` is control — nothing moves until you say so. Same contract, wildly different risk profile.

Hardware integration is not BGK's domain, you need conveyor and scanner hardware wired into its import interfaces, with the CogiScan Product Flow Controller available as an all-in-one box. And **PCB validation is not BGK's problem either**. BGK asks the question. Deciding whether a board may proceed is entirely the MES's job.

### BoardRequest Is a Synchronous Veto

`BoardRequest` is the whole reason Board Gatekeeper exists. The request is small:

| `BoardRequestData` | Meaning |
|:--|:--|
| `Board` | `Barcode` (empty string if nothing was scanned) and `BoardTime` |
| `Position` | Where in the line this happened |
| `Context` | A GUID correlating Request / Released / Failed for the same board |

`Context` is the field you build your tracking on. It is the only thing tying a `BoardRequest` to the `BoardReleased` or `BoardFailed` that eventually follows it.

The response is where the MES exercises control:

| `BoardRequestResult` | Effect |
|:--|:--|
| `RequestResult` | The verdict — see below |
| `Reason` | Free text, surfaced on error |
| `BoardPath` / `BoardSide` | Which board and side the result applies to — `Top`, `Bottom`, `Undefined` |
| `OverridingBarcode` | Rewrite the barcode and whisper the new one down the line |
| `RecipeName` | Tell the stations which program to run (Process Lens only) |
| `VIHResult` | Virtual Inkspot Handling — mark individual subpanels to be skipped |
| `BoardCorrection` | Per-placement delta on x, y, theta, per subpanel |
| `AdditionalBoardData` | Extra board and subpanel data for BGK |

And the verdict itself:

| `RequestResult` | What the line does |
|:--|:--|
| `Confirmed` | Board proceeds to the next processing step |
| `Rejected` | Board is locked in the conveyor and must not enter the next step |
| `Internal_Error` | Conveyor stops, board must be removed from the line |
| `PassThrough` | Board passes through without being produced |

Two things about this table. First, `Internal_Error` is not a soft failure — it is an operator walking to the line with a pair of gloves. Throwing an unhandled exception out of your validation logic is a production stop.

Second, everything here is a *string*. `RequestResult` is the string representation of `BoardNotificationResultValues`, and so is `BoardSide`, and so are the enums on `Position`. That was deliberate: serializing enums as strings means adding a new enum member in a later interface version does not break software built against the old one. It is a small, unfashionable decision, and it is precisely what lets one adapter serve five interface versions at once.

---

## OIB Eventing

Board Gatekeeper is request/response. Nearly everything else in OIB flows through Eventing — it is the central nervous system of the platform, and every adapter publishes into it.

OIB draws a sharp line between two kinds of adapter-initiated communication.

**Simple eventing** is for data of minor importance. Your client opens its own web service, the adapter calls it directly, and if a message is lost nobody notices. The canonical example is the Optimizer Adapter reporting progress on an optimization run. A dropped progress update is a progress bar that jumps.

**Safe eventing** — what OIB actually means by *OIB Eventing* — is for mission-critical data that is not allowed to get lost while the client is down. The Setup Center packaging unit consumption and movement. Miss one of those and your MES is planning production against material that no longer exists, which the docs describe, accurately, as causing "painful production problems".

![Subscriptions](https://image.j-roque.com/posts/20260908-oib/asm_eventing.png)

---

## Connect IoT

OIB is complex: many adapters, many configurations. It requires the user to create a .net framework application and integrate with each different type of adapter. This easily leads to bespoke applications that exist and spread inside your shopfloor.

With CM MES Connect IoT this is all seamless. Let's build a simple integration using OIB `Board Gate Keeper` for interlocking on Track-In and use `Eventing` to perform a trackout.

### Setup

In the setup we can configure the all the typical configuration of OIB Core access, like the sdk, configuration manager and address, the lines and sites we want to handle and the lines we want to ignore.

![Setup](https://image.j-roque.com/posts/20260908-oib/workflow_setup.png)

In the task `Extensions Setup` we can configure all the OIB extensions we want to use for our implementation.

![Board Gate Keeper Setup](https://image.j-roque.com/posts/20260908-oib/setup_bgk.png)

Here, you will have access to features like selecting where the Board Gate Keeper should be registered in the ISA95 tree and what are the port ranges it should use.

![Eventing](https://image.j-roque.com/posts/20260908-oib/setup_eventing.png)

For eventing we can set where in the ISA 95 tree to listen to. Crucially, we also can choose when we consider an event *stale* to be discarded. Important in reconnect cases.

### Material In

Sending the information to the MES querying if a particular panel can enter a machine via board gate keeper is very transparent. We use the `Gate Keeper Board Request` task to receive a request. We can then perform transformations and call the MES to signal we want to track-in a particular panel. The MES is already very robust in making sure that we only produce what we should produce and that everyting is in the proper conditions to produce.

If the MES finds any problem in the request it will reply back with an error. We will catch that error and reply back to the OIB, with a requestResult of Error and an error message that will be displayed in the HMI of the machine. If it's ok, we will simply reply back with a requestResult *Confirmed*.

![Material In](https://image.j-roque.com/posts/20260908-oib/material_in.png)

### Material Out

For material out we won't apply interlocking. 

>It's true that we could with the OIB *Traceability* module Board Produced Request, but this is not a common request. 

We will just be notified that the panel has finished by eventing and we will track-out the material in the MES.

![Material Out](https://image.j-roque.com/posts/20260908-oib/material_out.png)

---

## Modelling OIB with Connect IoT

In reality these kind of integrations are fairly uncommon.

The Electronics industry template, ships OIB support out of the box. It covers a plethora of adapters and MES scenarios [here](https://help.criticalmanufacturing.com/userguide/industrytemplates/smt/equipmentintegration/oib/).

For example, mapping the same scenario with the Electronics industry template (EIT) is just filling in a smart table.

![EIT Integration](https://image.j-roque.com/posts/20260908-oib/EID_Integration.png)

The workflow may seem complex, but it implements a full coverage of OIB use cases and adapters. It also supports different ways you can use OIB for all these MES actions. It also is resilient and robust and has already been battle tested against naive implementations of these integrations.

![OIB Workflow](https://image.j-roque.com/posts/20260908-oib/EIT_oib_Integration.png)

The EIT comes out of the box with typical integrations for Electronics ready to use, from Hermes, OIB, IPC-CFX, Fuji-Nexim, etc.

We can then leverage those integrations as they are or use extendable controllers to adapt them to our particular use case [here](https://j-roque.com/posts/20250730-iot-extensibility-iv/)

---

## Final Thoughts

OIB is a complex protocol, it's industry and vendor specific. But with CM MES Connect IoT and Industry Template it becomes as simple as configuring a table. It buys you the peace of mind of having a resilient, turn key solution to one of the hardest nuts to crack on an electronics shopfloor.

Right now, on some SMT line, a printed circuit board is sitting motionless in a conveyor while the line waits — because a web service has not yet said whether that board may continue. The MES receives this requests validates all is well, the recipe is correct, the line setup is ok, the machine is the correct one, the lot is as expected and it has the required raw materials and replies back with an ok, and the board can continue.