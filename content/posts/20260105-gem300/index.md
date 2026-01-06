---
title: "GEM 300 a Silent Revolution"
description: "GEM 300 a Silent Revolution"
summary: "GEM 300 a Silent Revolution"
categories: ["SECS/GEM"]
tags: ["Automation","Semiconductors"]
date: 2026-01-05
draft: false
authors:
  - Roque
---

This is a topic that I've been following for some time and haven't seen anyone talk about it. The impact of GEM 300 as a platform for a new breakthrough of solutions in semi.

## Overview

Semiconductor factories thrive on the seamless communication between manufacturing equipment and factory control systems (MES).

Historically, the `SEMI SECS/GEM` (SEMI Equipment Communications Standard / Generic Equipment Model) protocol provided a common framework for host-to-tool communication. 

Early implementations of SECS/GEM in 200 mm fabs were often loosely interpreted by different equipment vendors, each tool’s interface had its own quirks and custom behaviors. Integration engineers frequently had to create `one-off adaptations for each vendor’s GEM interface`, since each interface was custom to a particular equipment vendor and equipment type and even software revisions, making support and development time-consuming.

In other words, by the late 1990s and 2000s the industry still faced a lack of true plug-and-play interoperability, the need for a more rigidly standardized interface was becoming evident.

| ![Semi history](https://www.semi.org/sites/semi.org/files/styles/webp/public/inline-images/g33.png.webp?itok=fBd03e9r) |
|:--:|
| *Semi History* — Image source: *semi* |

## Traditional SECS/GEM Event Reporting Basics

In a standard SECS/GEM (E30) implementation, equipment notifies the host of important collection messages via events (typically S6F11).

Each `S6F11 Event` Report Send contains a *Collection Event ID* (CEID) and one or more associated reports carrying data values. The reports are defined by *Report IDs* (RPTIDs) which group specific *Status Variable IDs* (SVIDs) or other data variables to provide context for the event. 

The host can configure these via the GEM messaging scheme (e.g. S2F33 Define Report, S2F35 Link Event Report, and S2F37 Enable Event) to decide what data is reported for each event. This mechanism allows flexible linking of events to data.

For example, a “Lot Complete” event could be linked to a report containing SVIDs for lot ID, recipe, and yield. 

The S6F11 message structure remains consistent (CEID + list of reports) in all GEM implementations. 

By default, GEM provides only a generic framework: aside from a few required events, `most CEIDs and their meanings are defined by the equipment supplier`. 

For instance, the GEM standard requires basic material movement events such as a Material Received event when a carrier or lot is loaded onto the tool and a Material Removed event when it’s unloaded. However, the exact CEID numbers and any additional events (e.g. start/end of processing, operator interventions) were traditionally vendor-specific. `Integration engineers had to consult each equipment’s GEM interface manual to learn which CEIDs correspond to which real-world events and what data (SVIDs) would be provided`. 

In summary, traditional GEM offers the mechanism (S6F11 events and report linking) but not a standardized set of event definitions – each tool often had a custom list of CEIDs and reports for its unique capabilities. This meant the host’s event handling and report setup were tailored per equipment.

## The 300 mm Challenge and the Birth of GEM 300

The transition from 200 mm to 300 mm wafers around the year 2000 was a turning point for factory automation.

A 300 mm wafer (12 inch) is larger, heavier, and carried in an enclosed *Front-Opening Unified Pod* (FOUP) rather than an open cassette, making automated material handling almost mandatory. 

| ![Wafer transition](https://img.trendforce.com/blog/wp-content/uploads/2023/12/05102829/TSMC-Wafer-624x416.jpg) |
|:--:|
| *Wafer 200 mm and 300 mm* — Image source: *TrendForce* |

| ![Front-Opening Unified Pod (FOUP)](https://www.sps-international.com/content/net_products/8518-eWB0685-ASSY-1-eFOUP-2.jpg) |
|:--:|
| *Front-Opening Unified Pod (FOUP)* — Image source: *sps-international* |

Fabs investing in 300 mm lines aimed for “lights-out” automation, deploying overhead transport vehicles and robots (Automated Material Handling Systems, or AMHS) to move FOUPs between tools. 

This created new requirements: equipment needed standard ways to accept carriers automatically, verify the right lot has arrived, start the correct recipe, and report wafer-by-wafer progress, all with minimal human intervention. 

The basic SECS/GEM standards (SECS-II messaging [E5] over HSMS [E37] and the GEM protocol [E30]) were not sufficient alone to handle these advanced automation scenarios.

To meet this challenge, SEMI’s North America GEM 300 task force defined a suite of new standards – collectively nicknamed “GEM 300” since they were first implemented for 300 mm fabs. These standards augment the base GEM (E30) model with specific protocols for carrier handling, job management, and wafer tracking. Crucially, unlike the older 200 mm GEM implementations (which had a reputation for vendor-specific optionality), the GEM 300 standards were designed to enforce much tighter uniformity. Each standard in GEM 300 clearly prescribes when and how specific SECS-II messages must be used, defines state models for various operations, and mandates the events/data that equipment must report for nearly every action.

By the early 2000s, all major IC manufacturers were requiring 300 mm equipment to comply with this GEM 300 standard suite, making it a de facto universal interface for new fabs. 

| ![GEM300](https://www.semi.org/sites/semi.org/files/styles/webp/public/inline-images/GEM300-image-article-3.png.webp?itok=BTZlbs4_) |
|:--:|
| *GEM300* — Image source: *semi* |

GEM 300 builds upon the foundation of GEM. In 300 mm fab automation, the GEM 300 standards sit on top of the GEM (E30) and SECS-II communication stack, adding defined protocols for material handling, job execution, and tracking.

## GEM 300 Standards: A Unified Equipment Interface

GEM 300 isn’t a single standard but rather a `family of SEMI standards` that together ensure end-to-end automation control. 

These include the base GEM standard and several extensions introduced for 300 mm manufacturing. At a high level, GEM 300 covers the following functional domains: 
- Equipment Communication & Configuration (SEMI E5, E37 and E37.1)
- Carrier (material) handling (SEMI E84 and E87)
- Process job control (SEMI E40 and E94)
- Substrate tracking (SEMI E157)
- Performance/state monitoring (SEMI E10 and E116)

By standardizing each of these areas, GEM 300 essentially guarantees that all compliant equipment will exhibit the same interface behavior for equivalent actions, no matter the vendor. 

| ![GEM 300 SEMI Standards](https://www.agileo.com/sites/default/files/inline-images/GEM300_SEMI_Standards.png) |
|:--:|
| *GEM 300 SEMI Standards* — Image source: *agileo* |

Summary of GEM 300 functional areas and corresponding SEMI standards. GEM (E30) remains the core for basic messaging (alarms, data collection, remote commands, etc.), while new standards govern automated carrier transfers (E87), process and control jobs (E40 & E94), substrate (wafer) tracking (E90), and more. Together, these create a predictable, vendor-neutral interface for 300 mm tool integration

To understand how GEM 300 achieved *de facto* standardization, it helps to look at the key standards in this suite and what each one mandates:

- SEMI E30 – `Generic Equipment Model` (GEM): 

The foundation of all GEM 300 communications. E30 defines the common set of fundamental behaviors for any equipment interface – e.g. the connect/disconnect procedure, formats of status data, how to report events, how to upload/download recipes, etc. GEM 300 builds on this by requiring a `stricter set of standard events, alarms, variables`, and behaviors than were historically used, to ensure consistency across tools.

- SEMI E40 – `Process Job Management`: 

This standard formalizes how a host can define and control process jobs on the equipment. A Process Job is essentially a `recipe run for a specific set of material (wafers) on the tool`. It also allows the host to manage a job lifecycle with start, stop, pause or abort jobs via standard commands. By standardizing this, E40 ensures that recipe selection and execution are done in a uniform way: no matter the tool, the host can create a process job and be confident the equipment will interpret it (and report its status) consistently.

- SEMI E87 – `Carrier Management` (CMS): 

E87 addresses material handling, i.e. the loading and unloading of `wafer carriers` (FOUPs) at the equipment’s load ports. 

In practice, this means every GEM 300 tool adheres to the same sequence for the carrier interactions with the machine from arrival, carrier ID verification, latch/unlatch, and carrier removal. E87 defines detailed state models for load ports and carriers (e.g. states for when a port is occupied, ready, busy, out-of-service, etc.), along with events that announce each transition. Thanks to E87, a factory host can interface with any 300 mm tool’s load ports using identical commands and expect the same set of event notifications (carrier present, door open/close, mapping complete, etc.), greatly simplifying material logistics integration. Notably, this yields consistent material handling and eliminates vendor-specific “dances” that were common in earlier generations.

•	SEMI E90 – `Substrate Tracking`: 

While E87 deals with carriers, E90 standardizes tracking of `individual wafers` (substrates) within the equipment. Once wafers are loaded from a FOUP into a tool, the tool’s software must report each wafer’s location (e.g. which process chamber or slot) and processing state to the host in real time. This allows the factory software to know exactly which wafer is at which step, enabling lot-level and even wafer-level traceability. 

Essentially, E90 extends the GEM/GEM300 model down to the `single-wafer granularity`, using standard events for wafer movement and completion. Just as E87 did for carrier handling, E90 provides a uniform method for all tools to report wafer progress, so an integration engineer doesn’t have to worry that “Tool A calls it wafer ID X in module Y while Tool B uses a different convention” – they all follow E90’s method of substrate identification and tracking.

•	SEMI E94 – `Control Job Management`: 

E94 adds a higher level of coordination above the process jobs. 

A Control Job is essentially a container for one or more process jobs, defining a `sequence or batch of jobs that should be executed together` (for example, processing multiple FOUPs in a certain order, or splitting a lot across parallel chambers). 

Through standard messages, the host can create control jobs that queue up multiple process jobs, start or stop all jobs in the group, and specify post-process destinations (e.g. after processing, wafers from carrier A should move to carrier B). In effect, E94 standardizes batch and workflow management so that fab scheduling algorithms can orchestrate multi-lot flows in the same way on any GEM 300 tool. The benefit is more than just convenience – it enables advanced capabilities like lot buffering, sorting, and dynamic reordering using a common language. 

## Interoperability in a GEM 300 World

By introducing the above standards as requirements for 300 mm tools, the semiconductor industry effectively created a *de facto* standardized equipment interface. Today, any piece of equipment shipped to a leading-edge fab must support GEM 300 communications and this has profoundly changed the landscape of integration. 

The recent introduction of mandatory `Well-Known Names` allows also for a greater interoperability. In the past vendors could have names for events and variables that were not strictly defined, they could vary in casing, hyphenation or even similar names. This created a lot of entropy when an integrator tried to analyze and compare across specifications. With strict semantic conventions, this eliminates ambiguity, custom namings and narrows the range for interpretation. Each standard will now enforce a set of namings for all its components.

Let's see an example:

```bash
E90:ProcessJobCompleted
E87:CarrierArrived
E30:ControlStateChanged
```

For example for the *ProcessJobCompleted* this means that every specification is mandated to refer to this event as ProcessJobCompleted it can't be Process_Job_Completed, PJobComplete, or any other variation that introduces a need for interpretation.

Instead of fighting through custom protocols and vendor-specific GEM quirks, factory integration engineers now work with a well-defined, predictable interface that is remarkably similar from tool to tool. In fact, GEM 300 compliance is universal in 300 mm fabs, meaning a GEM 300-enabled machine can be deployed in any 300 mm front-end facility with minimal to littler or no customization by the host software. This plug-and-play quality was simply not achievable in the 200 mm era, where each new tool could mean weeks of custom interface coding.

Several factors make GEM 300 integration more straightforward and robust:

-	`Uniform Material Handling`: 

Thanks to standards like E87, the sequence of loading a FOUP is the same on every tool, the host always sends the same commands to request a carrier, and always receives the same key events (arrival, dock, mapped, etc.). This consistency guarantees behaviors and compatibility between load ports and carriers, regardless of equipment supplier, allowing for a unified global manufacturing ecosystem. 

For the fab, it means automated transport systems and stocker robots can interface with any tool’s load ports in a generic way. For integration engineers, it means far fewer special cases or custom scripts for carrier handling, the differences between vendors have largely been abstracted away.

-	`Standardized Recipes and Jobs`: 

With E40 and E94, the mechanism to select recipes and run jobs is standardized. A factory host can create a process job (E40) for Tool X or Tool Y and the procedure to start that job is identical. The host knows it can pause or abort the job via the defined GEM messages, and the tool will respond in a known manner. E94’s control jobs allow the host to manage multi-lot workflows in a uniform style across the fab. 

This has enabled fabs to implement advanced scheduling and dispatching systems that treat all tools consistently, since the interface for queuing jobs and moving wafers between carriers doesn’t change from one vendor’s tool to another. The net result is improved interoperability, equipment from different manufacturers can cooperate in a single automated flow because they adhere to the same job control semantics.

- `Comprehensive State Models & Data`:

GEM 300 forced equipment makers to expose much more of their internal state in a standard way. 

For example, beyond just the basic “equipment status” (which was defined in SEMI E10), GEM 300 standards define specific state machines for load ports (E87), for process jobs (E40/E94), and track each wafer’s state (E90). This means the MES can rely on getting detailed, `real-time data about what each tool is doing at every step`.

Every GEM 300-compliant machine will report events like *Carrier Ready to Unload*, *Job Started*, *Wafer Completed*, etc., using the same event IDs and formats. This predictability has made factory software much simpler, the integration layer can subscribe to a known set of standard events and be confident that if, say, “Lot ABC finished processing,” the tool will send the predefined E90/E94 events to indicate exactly that. Essentially, GEM 300 replaced ambiguity with explicit, standard messages and states.

## Final Thoughts

GEM 300 transformed the semiconductor equipment interface from a loose framework into a `predictable, standards-driven platform`. 

Factory integration engineers no longer have to `reverse-engineer each tool’s quirks or write extensive custom code` for basic operations, those are now covered by the GEM 300 “common language.” The result is a semiconductor fab where tools from any vendor can be integrated with far less effort and with confidence that they will “speak” identically in terms of carriers, jobs, recipes, and wafer tracking. This *de facto* standardization has enabled the highly automated 300 mm “smart fabs” of today, where material moves efficiently and processes are tightly coordinated by the host. In contrast to the past, when “each connection took far too much time to complete” due to variability, modern fabs enjoy a plug-and-play ecosystem – a testament to the power of the GEM 300 standards in achieving true interoperability.

For CM MES this allowed for a whole new way of addressing SECS-GEM as a communication protocol, we moved from tailored solutions made for each machine into an `adaptable template`. This has massively impacted implementation time and allowed us to partner with equipment vendors in order to have a streamlined path for interface and control.

The reality in 2025 is that a GEM 300-compliant interface is expected on semiconductor equipment, and it delivers a level of integration predictability that was simply not possible in the early days of SECS/GEM. This predictability and consistency continue to be crucial as the industry pushes toward even greater automation and Industry 4.0 capabilities.

---

Industry News, Trends and Technology, and Standards Updates | SECS/GEM (2)
https://www.cimetrix.com/blog/topic/secsgem/page/2

GEM300 Introduction | Agileo Automation
https://www.agileo.com/en/resources/gem300-introduction

GEM300 - 300mm SEMI Standard for Factory Automation
https://www.cimetrix.com/gem300

GEM300 SEMI Standards - PEER Group
https://www.peergroup.com/gem300-semi-standards/

SEMI GEM300 Standards - PDF Solutions
https://www.pdf.com/standards/semi-gem-300-standards/

SEMI E40: Process Job Management - PEER Group
https://www.peergroup.com/definition-of-standard/semi-e40/

SEMI E87 - Specification for Carrier Management (CMS) - PDF Solutions
https://www.pdf.com/standards/semi-e87-specification-for-carrier-management-cms/

SEMI E90: Substrate Management - PEER Group
https://www.peergroup.com/definition-of-standard/semi-e90/

SEMI E94 - Specification for Control Job Management - PDF Solutions
https://www.pdf.com/standards/semi-e94/

GEM300 Tutorial Overview | SEMI
https://www.semi.org/en/standards-watch-2021Dec/gem300-tutorial-overview

