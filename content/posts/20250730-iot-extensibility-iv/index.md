---
title: "Part IV - Extendable Controllers"
description: "Extendable Controllers - a path to reusability and extendability"
summary: "Extendable Controllers - a path to reusability and extendability"
categories: ["Extensibility"]
tags: ["IoT", "Customization", "11.2"]
##externalUrl: ""
date: 2025-07-30
draft: false
authors:
  - Roque
---

An integrated solution for extensibility.

[Part I](https://j-roque.com/posts/20250725-iot-extensibility-i/)<br>
[Part II](https://j-roque.com/posts/20250728-iot-extensibility-ii/)<br>
[Part III](https://j-roque.com/posts/20250729-iot-extensibility-iii/)

## Overview

**Part IV**, of the **IoT Extensibility series**, is dedicated to controller extensibility. Controller extensibility is a brand new feature of **MES version 11.2**.

As projects grow ever larger and more complex, particularly when we see projects across segments or with multi site strategies there are two mega trends.

### Templatization

The **first** is a trend for **templatization** of a segment feature set. 

In a scenario where shopfloors across multiple sites always have injection molding machines with similar processes. We can create a **template with a feature set** for injection molding areas. It can have features like custom user interfaces, custom logic and connect iot integrations. With templatization the user can structure a modular approach, where then each site can opt in and use that module or not. They can decide to include the template in whatever phase of their product they want.

![Templatization](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/templatization.png)

In this example, we can see that we have a pool of feature packages. The feature packages in this case are oriented to a particular process.

Let's imagine that **Site A** has an **Injection Molding Area** that will be addressed in the **first phase** of the project, the Site A will pull the feature package for Injection Molding Machines and have those features available. 

If we look at the **SMT feature package**, we can see that it is actually being used for **Site A** and was installed **phase II** and by **Site B** at **phase I**. This is a more flexible approach. 

These packages are themselves versioned and can suffer updates and development of new features. So  may depend on the SMT package at version 1.0.0 and later on update to version 2.0.0.

The **template solution** normally is a more **distributed approach** where we may have multiple sites with different teams all contributing to these modules and expanding them as they need.

### Baselining

The **second** trend is **baselining**, or golden model. 

This is very typical in a **multi-site** structure where all the sites are very similar between them. The user can then define a **golden baseline of how the factory should behave** with all the particular business logic and customization he may wish to add. 

The MES for all the sites will contain not just the CM MES, but also this additional layer of customization.

![Baseline](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/baseline.png)

The baseline may have their own particular team and architects and then **each site will depend on the baseline**, and as the baseline is updated they will also move to the latest versions of the baseline.

---

As you can see as MES implementations grow in scope and ambition, we now think globally and it generates new opportunities for abstraction and simplification.

**Extendable Controller** is a feature that comes as a **solution in IoT** to try and address the problems caused by **templating** or **baselining** for a controller integration.

## Trade-Offs on Abstracting

In both the the templatization and baseline there is a **common problem** that emerges, which is the need to **customize or extend the customization**.

Let's use the injection molding machine as an example. 

We can have a package with a feature set for injection molding machines with out of the box integration for particular machine types or vendors. It is often the case that in a **particular site**, there is the need to use the **template implementation** for the injection molding machine, but then require additional features. This without losing the relation to the template and an update path for the template.

The site specific customization could clone whatever implementation the template provides, but cloning implies that the **cloning entity loses all the relationship to the template**. After cloning your entity has a completely different lifecycle. What this means is that when the site updated the template and wanted access to new features, they would have to clone again and merge all their particular feature set.

![Template Clone](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/templateclone.png)

![Template Clone Update](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/templatecloneupdate.png)

There's also another downside, the user could then change the workflows he cloned from the original controller and would have **no way of going back to the original workflows**.

## Extendable Controllers

The new feature introduced in version 11.2, solves this problem. It creates a parent child relationship between controllers.

The child controller will then extends the parent controller workflows and is able to override them or add specific workflows.

## Baseline & Template Change

With Extendable Controllers, the user is able to have a **controller** that has a **parent-child relation** with the provided controller. This means that **any version change to the parent controller**, in non overridden workflows, **will automatically affect the child controller**.

![Parent Child](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/parentchild.png)

![Parent Child Update](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/parentchildupdate.png)

This allows a **separation of concerns** between **who manages the baseline** and the template and **who manages particular site specific customization**. 

In this example, we can see that the **Site Controller** added the **Workflow C**, when the parent controller is updated to a **new effective version** with a **new Workflow D**, automatically that change will be seen in the child controller, but its **Workflow C will remain unaffected**.

Let's imagine a new use case where the Site specific implementation has **overridden Workflow A**.

![Parent Child Overridden](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/parentchildoverridden.png)

![Parent Child Overridden Update](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/parentchildoverriddenupdate.png)

By being **overridden the Workflow A** of the **child controller takes precedence**.

The update will **add the new Workflow D**, but **Workflow A will remain the same** with the Site specific changes.

With this feature we have an integrated way to Extend Controllers without having to copy/clone them and creating a clear update path. Allowing for the teams that are building the feature sets and the teams that are building Site specific logic, to not diverge and create two very disparate sets of solutions, for very similar problems.

## Compounding

The parent child relationship are always **1-N**, one parent can have multiple children, but **a child controller can only have one parent**. But an interesting feature is that **a child can be a parent**. So we have an open door to controller compounding.

Let's imagine a cumulative example, where we have a controller for a Station. 

This station is managing the operator material actions. For particular stations, they have also some inspection results it needs to collect. Additionally, for some stations it also has integration with instrument equipment like scales or thermometers.

Typically, this would be very hard to abstract, you would need to have one controller that would have the driver to connect to the station, then a driver for the inspection machine and finally an additional one for instruments. 

It is often the case that with complex multi layer integrations the owners of each layer are not even the same. With the current solution you would be forced to have all of them working in the same implementation, with a controller that would control all the drivers required to model the process of the station with instruments and inspection. Another approach could be splitting into several controllers each only controlling a subset of the process, but then you would lose the big advantage of having a holistic approach to the process and the information and logic sharing between these integrations. You would also have no way to reuse these integrations in a modular approach, having to always start from scratch and map these processes.

Extendable Controllers implements a simple and intuitive solution for that problem. It allows the **compounding of parent child relations to map these variations**.

With this approach you could have a set of instances that use the controller for the station, another set that uses the controller with station plus inspection and even another set that uses station plus inspection plus instruments.

We can have the maximalist integration:

![Extendable Controller](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/extendablecontroller_rev.png)

Or a subset with only the Station integration and Inspection:

![Extendable Controller Subset](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/extendablecontrollersubset_rev.png)

This way we can have a **baseline or template** that delivers a **Station integration** and then the **site can add their own controller**, like the Inspection controller with additional features. This preserves the link to the Station controller and its update path, without sacrificing the ability to add new features.

![Baseline Controller](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/baselinestationcontroller.png)

## Use Case

Let's map this as a use case. We will use the compounding example and add workflows to our controllers.

In our use case, the structure is that we have an Automation Controller child that has two workflows, then two more workflows that come from his parent the Inspection Controller and finally two more that come from the grandparent controller, the Station Controller.

![Use Case Extendable Controller](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/usecaseextendable.png)

Let's imagine that now, in your child controller, you wish to **change the Material Tracking workflow** of your parent. For example, you want to add logic if any measure is out of bounds the Station TrackOut must also put the material on Hold.

![Use Case Extendable Modify Workflow](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/usecaseextendablemodifyworkflow.png)

We will **override the workflow** to add this logic. By overriding a workflow from a parent or grandparent you are effectively **disabling the workflow of the parent** and **pulling it into your child controller**.

![Use Case Extendable Override Workflow](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/usecaseextendableoverrideworkflow.png)

Then you can **add your own logic to the workflow**. If you wish **you can later on revert it**, by reverting you will cease to have that workflow in the child controller, but will have it as a reference to a parent controller.

## Use Case - Using the MES

Let's see it in the MES. 

After creating a **Station Controller** and setting it as **effective** we can now create our **inspection controller that extends the station**.

When we create our Inspection Controller we can create our first parent child relation.

![Inspection Controller](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/controllerextendedinspection.gif)

In the Inspection controller we can choose to add or not its own driver definition. In our use case we wish to add a driver definition responsible to collect information from files generated by the inspection machine. The **workflow view will show all the workflows that are active** and will be used in the integration.

We will have two workflows (Setup, Material Tracking) that are inherited and two that are from our child implementation (Inspection_Setup, Inspection Result).

We can now add our final layer, we can add a Controller focused on our Instruments integration.

![Instruments Controller](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/controllerextendedIInstruments.gif)

This controller will have as parent our Inspection Controller. By **adding the Inspection Controller as parent**, it will now have **access to all the workflows** the Inspection Controller had, including the one's that came from his parent.

The Instruments controller is now a compounding of `Station->Inspection->Instruments`.

![Instruments Override Controller](https://image.j-roque.com/posts/202507230-iot-extensibility-iv/controllerextendedIInstrumentsoverride.gif)

In the instruments controller, we are now free to use all the available workflows. We can even override workflows that were inherited. By **overriding** we are **creating an editable copy of the original workflow**. 

We are always able to revert to the parent workflow. This is very helpful when we need to perform some specific logic that is later provided by our parent controller. We can just revert back to the new version of the parent workflow.

Also, we have a native integration between child and all its genealogy. When in a workflow of a child controller, with a simple button press the user is directed to the view of that workflow in its original controller.

## Final Thoughts

This 11.2 feature constitutes a milestone in providing a simple and effective way to scale and adapt to ever more complex working modes and solutions. 