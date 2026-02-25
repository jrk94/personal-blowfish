---
title: "Relational Anchor System"
description: "From hierarchical to relational a journey on your anchor system"
summary: "From hierarchical to relational a journey on your anchor system"
categories: ["MES"]
tags: ["MES","Anchor","Graph"]
date: 2026-02-25
draft: false
authors:
  - Roque
---

Why do organizations need an anchor system for an AI strategy and why that system must have not just a hierarchical perspective, but also a deeply relational one.

## Overview

Most of your systems exist in silos. They can have an internal coherence and semantic but that semantic does not cross over to other systems.

An ERP speaks ERP, a SCADA speaks SCADA and an MES speaks MES. This has been an age old problem of system interoperability.

This was a very hard wall to bring down, because it actually matched the user base silos. ERP users typically only use the ERP, therefore for them it's fine to create their own jargon and semantics. A dialogue between different silos is inherently hard as it will be flooded with specific nomenclatures and codes, that only people in the silos are able to make sense of.

| ![Tower of Babel](https://upload.wikimedia.org/wikipedia/commons/thumb/f/fc/Pieter_Bruegel_the_Elder_-_The_Tower_of_Babel_%28Vienna%29_-_Google_Art_Project_-_edited.jpg/1280px-Pieter_Bruegel_the_Elder_-_The_Tower_of_Babel_%28Vienna%29_-_Google_Art_Project_-_edited.jpg) |
|:--:|
| *Tower of Babel* — Image source: *https://en.wikipedia.org/wiki/Tower_of_Babel* |

>AI has turned what used to be an interoperability headache into a strategic suicide.

AI is boosted by having a large knowledge base. The bigger the knowledge base, the bigger the inference capacity and accuracy. 

Information silos and babilonic semantics are the bane of AI.

If depending on the interface, a product is a sku (Stock Keeping Unit), or a container is a FOUP (Front Opening Unified Pod), this means that AI won't have the inference capacity to make relations between those different information sets, that actually refer to the same shopfloor object.

AI needs a **live structured and coherent dataset**, that is a representation of everything happening on your enterprise.

## Anchor System

The solution is finding your **anchor system**. This was made really clear by [Jeff Winter](https://www.jeffwinterinsights.com/)'s presentation at [ProveIt 2026](https://www.proveitconference.com/).

The goal is clear, organizations need to find what system drives their business and therefore which system is their semantic anchor. 

This is a system that percolates and drives your organization. It provides a perspective where you feel comfortable to homogenize your organization with.

But how to chose? What should be your anchor system?

If your business is driven by being a part of a constrained supply chain or by answering to purchase orders as fast as possible, your ERP system could be a bet. Or if your enterprise is highly automated with low product mix, where your goal is just to keep it running as long as possible maybe your scada system is your bet.

For most mature businesses, the `holistic nature of the MES` as your shopfloor management and execution system, makes it the natural anchor system. It's the one forced to interface with all other systems and the one with the more heterogeneous user base. 

From operators and line managers to quality engineers, machine engineers, and plant managers, they all interact daily with the MES. It is where they go not only to find answers, but to uncover relationships between events on the shop floor.

Contrary to other systems, MES does not have a myopic view of the shopfloor, but it encompasses all the actions performed in and around your shopfloor. For more specialized software, the MES interfaces with it and provides and extracts all the relevant content.

It holds all the semantic context and nomenclature of all the objects in your shopfloor. It has a view not just of hierarchy but of dynamic relations.

## Relational System

Most systems and strategies try to standardize on a *hierarchical topology* of the shopfloor. The process is a pain and becomes very complex, because the systems are intrinsically passive and ambiguous. They expect a third-party to give it it's place on the hierarchy. 

Notice, how this is an oversimplification of the dynamics of the shop floor. A resource name, by itself, provides insufficient context on the actions and role it is performing in the shopfloor. What is a process resource today may be running quality samples or R&D products tomorrow. The resource and its ISA95 structure provides little context on its real time role in the shopfloor.

The **MES is a control system**.

It does not only define topology, it defines an **opinionated ontology**. It encodes a perspective on how entities exist, how they relate, and what is valid. It eliminates ambiguity. It enforces behaviors and business rules.

This necessarily constrains the degrees of freedom on the shopfloor. That is not a side effect, it is the point. An MES prevents unconstrained action by design.

In languages, what LLMs were trained with, It happens the exact same thing. There are words, sentences and semantic structure. There are degrees of freedom in creating new words, but the overall semantic guardrails stay there all the same.

Of course, if your MES is insufficient, either in the user experience or in missing key functionalities, this becomes a pain. You end up creating grey zones or black holes in your factory. This is one of the key reasons, it is important to chose a mature product with a *data model that is able to map your shopfloor as closely and with as much control as possible*.

Some architectural patterns like `UNS` (Unified Namespace) are a very interesting way to be able to quickly assign a hierarchy in your shopfloor and be able to start creating a semblance of data model and data structure. Nevertheless, they suffer from a **lack of a relational context**. 

In a previous [blog post](https://j-roque.com/posts/20260218-frominsighttoaction/#why-context-is-everything) we explained how in an AI dialog with the MES we were able to quickly explore and discover our shopfloor:

{{< include-local file="Generate a complete relational graph of Site Europe-West.html" >}}

One of the key aspects of this node diagram is that it goes **beyond the ISA95 tree**. We see concepts that are not a hierarchical place, but a functional association.

We map not only physical objects, but also our process in the form of flows and steps.

A flow is a process sequence that defines how materials move through the manufacturing system, so they are vital in mapping how the materials are moving through the shopfloor. A step is an individual operation or stage within a flow. A step requires a service to be performed and a resource may provide multiple services. 

We now see that our shopfloor is actually much more dynamic than just `Enterprise, Site, Facility, Area, Resource`. It actually has complex and deeply relational interactions between objects. The **anchor** system of all those actions is the MES. 

If your system has no notion of these relations your **AI strategy will fail**, because AI will have a **monolithical and siloed vision** of what is happening in the shopfloor. 

It will miss all the relations in your shopfloor, from what recipe was used, raw materials, samplings, maintenance activities, non conformance, all that rich feature set that makes an MES and is the day to day life of your factory.

It will also not know if what is seeing is a deviation or if it's an expected action. The **pure ISA95 based systems are reporting and monitoring**, but fundamentally they are **not control systems**. They offer no explanation of why a particular recipe or raw material was used and if it should have been used. 

This means, your AI will always be working with polluted datasets.

>AI follows the age old adage in software engineering, **garbage in; garbage out**. 

If its datasets have to be cleaned or parsed, in order for them to have meaning, you already lost. 

No one will scrape petabytes of data to make sure they are conformant. 

You will end up in the current industry 4.0 roadblock. 

Massive unstructured datasets, from machine logs, test reports, divergent systems and no knowledge. 

You will have a massive pile of worthless data. And the pile will keep adding up.

## Final Thoughts

You can go bottom up, your machines dictate your data structure. But all machines are different and behave in a different way with a different structure and every machine process and topology are completely different. 

You can go top down, your ERP gives all your semantic structure. But what do ERP's know about shopfloor reality and constraints?

You only have one viable choice, put your house in order with an MES and chose an MES that already gives you all the rest out of the box.

That's why the **MES is your shopfloor anchor system**, because it's what captures your shopfloor structure and logic.