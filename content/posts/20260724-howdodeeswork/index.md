---
title: "Understanding Dynamic Execution Extensions (DEEs): How They Really Work"
description: "A behind-the-scenes look at how DEEs execute inside Critical Manufacturing MES — Action Groups, pre/post timing, caching, transactions, and the best practices that keep implementations maintainable."
summary: "Most engineers learn to create a DEE long before they understand the execution pipeline underneath it. This is what that pipeline actually looks like — and why it matters once a project gets big."
categories: ["Engineering"]
tags: ["MES", "Critical Manufacturing", "DEE", "Extensibility", "Software Architecture"]
date: 2026-07-24
draft: false
authors:
  - Roque
  - Ribau
---

Most developers learn how to create a **Dynamic Execution Extension** (`DEE`) long before they understand how the execution pipeline underneath it actually works. That gap doesn't matter on a small project. It matters enormously once a project grows: it starts showing up as performance issues, duplicated logic, Action Groups picked by guesswork, and debugging sessions that end with someone staring at a stack trace error they've seen a dozen times but never actually understood.

## Overview

This blog post is about what happens behind the scenes: why DEEs exist, how the CM MES product code actually finds and triggers them, how to choose between `Pre` and `Post`, `Operation` and `Orchestration`, and the handful of best practices that separate a maintainable implementation from a production incident waiting to happen.

## Why DEEs Exist in the First Place

When Critical Manufacturing built its MES, the team made a deliberate choice not to ship a closed system — the kind where the only way to add logic is to copy the product source and modify it. That approach might work in the short term on a fully managed CM Project, but it's not sustainable: you lose upgrades, you duplicate everything you touch, and you end up maintaining a fork of the product forever.

`DEEs` — Dynamic Execution Extensions — are one of the main answers to that problem. They give implementation teams, and power users, a way to extend and adapt the product without touching product code and without depending on Critical Manufacturing for every new requirement. A client with enough in-house knowledge can build their own logic and become genuinely independent. That's the point of the mechanism, and it's worth keeping in mind every time you're deciding *how* to use it — because a mechanism built for extensibility can just as easily be used to build something unmaintainable.

## Where DEEs Fit in the Request Pipeline

Strip away the specifics and a standard MES method call is simple: a request comes in, it flows through the method, and a result comes out. `DEEs` insert themselves at exactly two points around that flow. Before the method runs, the pipeline checks whether there's anything registered to execute first, if so, runs it. 

The standard method flow executes. 

Then, after the method completes, the pipeline checks again for anything registered to run afterward, executes it, and only then returns the output.

That's the entire mental model. 

Everything else — naming conventions, Action Groups, Pre versus Post — is just detail on top of this one idea: a hook before, a hook after, and the standard method in between.

{{< img src="https://image.j-roque.com/posts/20250821-howdodeeswork/MESRequest.png" alt="MES Request" whitebg="true" >}}

<br>

{{< img src="https://image.j-roque.com/posts/20250821-howdodeeswork/DEEsHook.png" alt="DEE Hooks" whitebg="true" >}}

## Naming Your Action Group: The Convention Nobody Tells You About

The Action Group name isn't **arbitrary**, and it isn't something you have to reverse-engineer from the product source. It follows a simple and fixed pattern and is described in the documentation:

```
{NamespaceSuffix}.{ClassName}.{MethodName}
```

Take the `CreateMaterial` method as an example. Its .NET namespace is `Cmf.Navigo.BusinessOrchestration.MaterialManagement`, its class is `MaterialOrchestration`, and the method is `CreateMaterial`. Strip the namespace down to its suffix — `MaterialManagement` — and combine it with the class and method names, and you get:

```
MaterialManagement.MaterialOrchestration.CreateMaterial
```

Append `.Pre` or `.Post` depending on where you want your logic to run, and that's your Action Group. It's a mechanical process, not magic, it's true because of retro-compatibility, a handful of older methods don't follow this convention to the letter, but those are documented exceptions.

You don't have to know this by heart, there's a reference for exactly this: the [Extension Points documentation](https://developer.criticalmanufacturing.com/reference/api-extensionpoints/index.html) lists every extension point available in MES. It's a resource that, according to the training session feedback, far fewer developers know about it than they should.

## The Magic Behind the Curtain: StartMethod and EndMethod

Knowing the naming convention answers *what* your Action Group is called. It doesn't answer how the product code actually finds and executes your DEE. That happens inside two utility calls that wrap the standard method body:

```cs
namespace Cmf.Navigo.BusinessOrchestration.MaterialManagement
{
    internal partial class MaterialOrchestration : IMaterialOrchestration
    {
        public CreateMaterialOutput CreateMaterial(CreateMaterialInput createMaterialInput)
        {
            _utilities.StartMethod(objectTypeName, "CreateMaterial",
                new KeyValuePair<String, Object>(nameof(CreateMaterialInput), createMaterialInput));

            // Method code…

            _utilities.EndMethod(createMaterialOutput.Material.EntityType.Id,
                createMaterialOutput.Material.Id,
                new KeyValuePair<String, Object>("CreateMaterialInput", createMaterialInput));

            return createMaterialOutput;
        }
    }
}
```

`StartMethod` builds the Action Group name and appends `.Pre`; `EndMethod` builds the same name and appends `.Post`. 

Each call checks whether any DEE is registered against that resolved name, and if so, executes it. That's the whole mechanism, there's no separate dispatcher, no background listener. It's these two calls, wrapping the method body, doing the lookup and triggering the DEEs synchronously as part of the same call stack.

## When Multiple DEEs Share an Action Group

There's nothing stopping you from registering as many DEEs as you want against the same Action Group, and in real implementations that can happen, a single `TrackOut.Post`, for example, might have several DEEs attached to handle different pieces of logic. 

When that happens, execution runs in a **cascading sequence, lowest order to highest**.

That guarantee only holds if you actually set distinct orders. Leave two DEEs at the same order — which happens more often than you'd think, especially when master data gets bulk-loaded — and the execution order between them is undefined. You don't get to choose which one runs first, and if one depends on state the other mutates, that's a real bug waiting for the wrong day to show up.

There's a second, subtler trap: a DEE's position is relative to the Action Group, not absolute. The same DEE that runs third in one Action Group can run first in a different one it's also attached to. If that DEE assumes something was already done by "the DEEs before it," that assumption silently breaks the moment it's reused somewhere else.

And then there's the **plain performance math**. If ten DEEs are attached to `TrackOut.Post` and each takes a 300 milliseconds to run, you've added three seconds to every track-out — permanently, not as a one-off cost. On a line producing a unit every couple of seconds, adding seconds to track-out isn't a rounding error, it's a bottleneck that turns into a production down event.

The reflex response, merge all ten DEEs' code into one, doesn't help a lot. The code is the same code, running the same instructions; consolidating it into a single DEE buys you little in runtime and costs you readability and debuggability. 

Try and reduce chaining, either vertically (in the same action group) or horizontally (hooking both in the service and in operations called by the service). Be simple and obvious, smaller, single-purpose DEEs are easier to understand, easier to debug, and easier to reorder safely later.

![DEE Hierarchy](https://image.j-roque.com/posts/20250821-howdodeeswork/DEEHierarchy.png)

## Pre or Post? Fail Fast, Then Act

The decision between `Pre` and `Post` usually comes down to one question: 

>Does this logic depend on the operation having already happened, or does it need to stop the operation from happening at all?

Take rework. If a DEE needs to send a material to a rework flow *after* it's been processed, it has to run on `Post`. It wouldn't make sense to send the material to rework before the track-out that triggers the rework in the first place.

Now flip it: you want to block a track-out entirely if the material is missing a required attachment. That validation has to run on `Pre`. You could technically validate after the track-out and roll everything back on failure, but that wastes a full operation's worth of resources and time just to fail. The entire point of a validation is to **fail fast**, before you've done any work you'll have to undo.

That generalizes cleanly: validations belong at `Pre`. If you find yourself putting a process validation on `Post`, that's worth a second look.

> All the validations that you do should be at the pre of any Action Group.

## Operation or Orchestration? It depends...

This is the question that generates the most contention in practice.

The mechanical difference is straightforward. 

An `Operation` — for example `BusinessObjects.MaterialCollection.TrackIn` — does only the essential, object-level work, and it's *guaranteed* to run every time that operation executes, no matter what is the service calling it. 

An `Orchestration` — for example `MaterialManagement.MaterialManagementOrchestration.ComplexTrackInMaterials` — wraps several operations together and triggers everything around them: data collections, checklists, and whatever else the UI action is supposed to kick off. Orchestrations are usually what UI buttons call through their underlying services; they are *not* guaranteed to run if a lower-level operation is invoked directly, by a different orchestration, or by another DEE.

The rule of thumb: If you want a system wide impact default to `Operation`, if you want to pinpoint the change to particular API call or behavior choose the specific `Orchestrations` you want to impact. Take not that when making system wide extensions, they will impact the whole system and as such you need to be extra carefully validating this. Also, it's important to understand what is the granularity of context needed for your extensibility, that may also impact this decision. 

Where it gets genuinely contested is *where inside the orchestration lifecycle* to hook in when orchestration-level context is actually needed. Hooking at end of the orchestration as the requirement allows — triggering on `Post` at the orchestration level rather than reaching into an operation in the middle of it can ensuring that the standard way the API works will be kept as most as possible.

The counterargument, is that hooking into an operation mid-orchestration and then manipulating an object there is a well-known way to hit `"the object has changed"` errors, because the DEE mutates something that isn't part of the input being passed back up the call chain, and the reference the orchestration expects gets silently invalidated. Debugging that is its own kind of forensic exercise: reconstructing which operation, inside which orchestration, touched what, in what order.

There's also a performance angle: calling an orchestration *from inside* a DEE — instead of calling the underlying operation directly — adds the orchestration's overhead every time that DEE runs. The trade-off cuts both ways: skip the orchestration and call the operation directly, and you might silently lose whatever the orchestration was responsible for triggering. A concrete example from the session — calling the `TrackIn` operation directly, instead of going through the orchestration, means the data collections and checklist instances that the UI flow normally opens simply never get created.

None of this resolves into a single rule. It depends on what you're trying to guarantee, and what you're willing to skip.

## The Documentation Trick: Trace What an Orchestration Actually Calls

The technique behind that worked example is worth calling out on its own, because it's underused. The [Extension Points documentation](https://developer.criticalmanufacturing.com/reference/api-extensionpoints/index.html) doesn't just list Action Group names — for orchestrations, it shows what they invoke underneath, recursively. Instead of reading product source to reconstruct a call graph by hand, you can search the orchestration the UI calls, see the Action Groups available on it, and follow the chain down through every operation and sub-orchestration it triggers.

That lets you make an informed trade-off: the topmost orchestration usually does the most (and costs the most), and each layer down sheds functionality you may or may not need. Pick the shallowest layer that still gives you the context your DEE actually requires — nothing more.

## Anatomy of a DEE: Test Condition, Action Code, and References

Every DEE Action has a **Test Condition Code** block that must return a boolean, and an **Action Code** block that runs if that condition is true. 

Inside the action code, the `UseReference` directive takes two string arguments: the assembly to reference, and the namespace within it to use — the DEE equivalent of adding a reference in Visual Studio plus a `using` statement in C#. 

Two things worth knowing here that aren't obvious from examples: product assemblies don't need to be referenced at all — they're already loaded — and project-specific assemblies only need to be declared once, even if you use types from that assembly across many lines. Declaring the same reference three times doesn't multiply anything at runtime; it's just inherited copy-paste noise. It's also entirely valid to skip declaring the namespace and use the fully-qualified type name inline instead — slightly more verbose per line, but it keeps the top of the file clean, which is exactly what the auto-generated DEEs behind Business Workflows do (more on that below).

![DEE Structure](https://image.j-roque.com/posts/20250821-howdodeeswork/DEEStructure.png)

## Triggering DEEs: Automatic, Manual, and by Code

DEEs trigger three ways. 

- **Automatically**, by being associated with an Action Group or with a Rule attached to some other MES entity. 
- **Manually**, through the UI, where you pick the DEE, supply parameters, and execute it directly. 
- And **from code**, when you need to invoke one explicitly:

```cs
var serviceProvider = (IServiceProvider)Input["ServiceProvider"];
Cmf.Foundation.Common.Abstractions.IAction deeRule =
    serviceProvider.GetService<Cmf.Foundation.Common.Abstractions.IAction>();

deeRule.Load("CustomDEEAction");

List<KeyValuePair<string, object>> parameters = new List<KeyValuePair<string, object>>();
if (this.Material != null)
{
    parameters.Add(new KeyValuePair<string, object>("Material", this.Material));
}

deeRule.Execute(parameters);
```

## Other uses

DEEs aren't confined to Action Groups. They surface throughout MES: Checklists, Event Rules, Timers, Smart Table validations, Label Printing, Future Actions, Sort Rules, Maintenance Plans, Business Workflows, Automation Scheduled Action, and more. If you've built a Business Workflow, you've already created a DEE without necessarily realizing it.

That last one is worth seeing directly. Configure a workflow that, say, automatically moves a material to the next step whenever it changes into a specific step name — and MES generates the DEE for you behind the scenes, wired up to all the relevant Action Groups automatically. Open the generated code and you'll notice it skips `UseReference` entirely: everything it touches is product assembly and product namespace, so there's nothing to declare. It's a useful reference for what "clean" DEE code looks like when you don't need any project-specific references at all.

![Business Workflow](https://image.j-roque.com/posts/20250821-howdodeeswork/BusinessWorkflow.png)
![Business Workflow DEE](https://image.j-roque.com/posts/20250821-howdodeeswork/BusinessWorkflowDEE.png)

## Classification: Be specific

The classification field is easy to overlook, plenty of engineers never touch it, but it matters the moment a customer is autonomous enough to build their own Rules. Tag a DEE with a free-text classification, say `FreeWork`, and then scope a Rule to only display DEEs with that same classification. When the user goes to build the rule themselves, they only see the options that are actually relevant to it, instead of scrolling through every DEE in the system looking for the right one. It's a small feature, but it's the difference between "the user can safely self-serve" and "the user picks the wrong DEE because nothing stopped them."

There are also classifications that are important like, for executing DEEs from ConnectIoT they need to be with that scope.

## Behind the Scenes: Versioning, Storage, and Caching

Every save to a DEE creates a new version rather than overwriting the last one. 

That gives you **rollback**, **comparison between versions**, and **history**, a clear answer to "who changed this, when, and what exactly changed". A question every implementation team has had to answer under pressure at some point. Setting a different version as effective is a straightforward state flip between records; nothing destructive happens to the versions you're not using.

![Comparison DEE](https://image.j-roque.com/posts/20250821-howdodeeswork/ComparisonDEE.png)

Underneath, DEEs live in their own database schema, aptly named `DEE`. The core table, `Dee.T_Action`, holds one row per version: the DEE's name, description, the action code itself, the assembly name it compiles to, the validation code, and the validation assembly name. Every version of every DEE, product or custom, it makes no distinction, lives in this same table and goes through the same caching pipeline.

![DEE Table Schema](https://image.j-roque.com/posts/20250821-howdodeeswork/DEETableSchema.png)

Loading works differently than most engineers assume. Every DEE is loaded into cache when the host starts, not lazily, all of them, up front. When someone changes a DEE's effective version or its Action Group associations, the cache detects the assembly name has changed and reloads just that entry. What actually sits in cache is exactly what's in the database table: name, action code, validation code, effective version. No compiled artifact yet.

The DLL only gets compiled — and written to a temporary folder inside the host — the first time the DEE actually *executes*, not when it's imported and not when master data is loaded. This answers a question that came up directly in the session: deploying a package that reinstalls every DEE version, even ones that haven't functionally changed, does not multiply the number of DLLs sitting around. You accumulate rows in a table; you don't accumulate compiled artifacts until something actually runs.

## DEEs vs Services: Choosing the Right Tool

This is the recurring architectural fork: extend the product through a DEE, or build a proper service. Both are valid, and the right choice depends on scale and who needs to touch the logic afterward.

| | **DEEs** | **Services** |
|---|---|---|
| **Pros** | Only way to extend product services at all; highly reusable for small blocks; enable/disable straight from the UI; flexible updates without a new package delivery; easier to debug on the client side | Better for large or complex codebases; cleaner for end users (logic hidden from the UI); less prone to unsynced client-side edits; easier to track execution history; supports service-level security |
| **Cons** | Client-side changes can get lost if not synced back to the repository; not ideal for large or complex logic; performance overhead on first execution; harder to track execution history; no service-level security for individual DEEs; vulnerable to breaking silently if the product renames namespaces or classes | Requires a new package delivery for every change; harder to debug directly on the client side |

The practical middle ground the session landed on: DEEs are the only lever you have for extending product services, for that use case there's no real decision to make. For everything else, size and ownership decide it. Large, structured logic that benefits from real code organization belongs in a service. Small, toggle-able, client-adjustable logic belongs in a DEE.

## Why You Can't See DEE Execution Time in Service Performance

This is the practical cost of the DEE/Service trade-off that catches people off guard: history tables tell a very different story depending on which one you used.

| | **DEEs** | **Services** |
|---|---|---|
| Service History entry | `Execute Action` — the same name every time, regardless of which DEE ran | The actual service name (e.g. `TrackInMaterial`, `ComplexDispatchAndTrackInMaterial`) |
| Where to find real detail | `dbo.T_OperationHistory` | `dbo.T_ServiceHistory` (already shows it) |

Because every DEE shows up in `T_ServiceHistory` as a generic `Execute Action`, the history UI will show as a ExecuteAction which you will then have to drill down to check what was the DEE action called.

The **Service Performance** report, which without observability can sometimes be helpful to out what are the services taking with a worse performance, is blind to what DEE action is the actual bottleneck. The user will have to the go into `T_OperationHistory` and join the two tables to get a real answer. That's a genuine operational disadvantage of DEEs worth knowing about before you lean on one for something performance-sensitive.

![DEE History](https://image.j-roque.com/posts/20250821-howdodeeswork/DEEHistory.png)
![DEE History2](https://image.j-roque.com/posts/20250821-howdodeeswork/DEEHistory_2.png)

## Best Practices That Actually Matter

### Try/Catch Has Limits You Don't Control

Catching an exception inside a DEE and quietly moving on works fine, right up until the method you called wraps its body in `StartMethod` / `EndMethod`. 

Those two calls maintain an internal stack of every start and end call made during the transaction. Swallow an exception thrown from inside that wrapped method, and the stack ends up with more starts than ends. That mismatch is exactly the cryptic error message a lot of engineers have hit without ever understanding where it came from.

You can catch and re-throw a different, friendlier exception, that's fine, because the transaction still fails the way the product expects. What you can't do is catch and *suppress* an exception coming out of a product method that uses `StartMethod`/`EndMethod`, and expect the transaction to close out cleanly. The validation happens at the end of the transaction, and it will catch the mismatch even if your code didn't.

![DEE Throw](https://image.j-roque.com/posts/20250821-howdodeeswork/DEE_Throw.png)

### Transactions Don't Roll Back the Outside World

The assumption "if something fails, everything gets rolled back" is true only for what's inside the database transaction. Anything that already crossed a system boundary is permanent the moment it happens, transaction outcome notwithstanding.

A DEE on `CreateMaterial.Post` sends a message to an ERP system, then goes on to move the material to another step. If that move fails, the material creation rolls back, but the ERP already received the message. Create the material again and you'll send that message a second time. If the ERP isn't built to deduplicate, you now have two systems out of sync because one of them thinks something happened twice. The same happens when you send a message bus message to Connect IoT. The machine will receive the message. If the transaction aborts, the message will have already be received by the machine.

> Changes are rolled back due to failure, except the ones that already crossed a system boundary. Everything inside the transaction is undone; everything the outside world already saw stays true.

Where possible, perform the irreversible, externally-visible action *last* in the sequence, so a failure means nothing external happened yet. Where that's not possible — timers, integration entries, anything asynchronous by nature — agree on a message format with a unique identifier the receiving system can use to ignore duplicates.

### Keep Your Output Objects Thin

The output object a DEE returns gets serialized into JSON at the end of execution. If that output includes an entity like `Material` with every property loaded — product, steps, facility, area, calendars, all of it — you're paying for that JSON size whether or not the caller needs any of it. It compounds: loading a fully-populated entity is also extra database round trips you may not need in the first place.

The fix is simple, load only the properties your DEE actually needs, and if you need a richer object to work with internally, build a separate object for that and keep it out of the output entirely. A good developer is a minimalist.

## Final Thoughts

The tutorials and documentation get you going, but some decisions take years to come back to bite you. In this blog posts we tried to check the major hurdles and pain points and also performing a comparative analysis.

After this you are ready to be even better at extending the MES system.

> This blog post was based on a talk by Nicole Ribau in 2025-08-21 @CM-Portugal