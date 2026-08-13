---
title: "Understanding UI Pages: How They Really Work"
description: "A behind-the-scenes look at UI Pages in Critical Manufacturing MES — Builder concepts, converters, presets, Business Workflows, and the pagination bug that shows up in almost every project."
summary: "Anyone can drag a widget onto a UI Page. Far fewer people wire it so it survives contact with 10,000 real records. This is what a UI Page actually is, and the mistakes that show up in nearly every implementation."
categories: ["Engineering"]
tags: ["MES", "Critical Manufacturing", "UI Pages", "Extensibility", "Angular", "Software Architecture"]
date: 2026-07-24
draft: false
authors:
  - Roque
  - Novo
---

UI Pages a low code designer to build your MES cockpits.

## Overview

We will address what a **UI Page** is made of, how its pieces connect, and through a live build of a real page where implementations quietly go wrong. First, a tour of the vocabulary (Builder, Widgets, Tasks, Properties, Links, Converters, Layouts), followed by a from-scratch build of a two-level grid, and closing with a set of best practices earned the hard way, on real projects.

## The Four Kinds of UI Page

A **UI Page** is a configurable entity, built from widgets, properties, and data sources, that displays information and reacts to user input and system triggers. 

MES supports four shapes:

- **Cluster** — a navigation panel on the left, a detail panel on the right. Think of the Step View or the Material view: click an item on the left, its details render on the right.
- **Page** — the most common type. A blank canvas you populate by dragging and dropping elements to build interaction.
- **Wizard** — a sequence of steps that collects and prepares data before handing it off to a single transaction service call.
- **Step** — a custom, reusable unit you build once and drop into any Wizard.

## The Vocabulary of the Builder

The **Builder** is the design surface itself: a palette of widgets on one side, a canvas to drop them on, and a mechanism — **Links** — to wire them together.

![Builder](https://image.j-roque.com/posts/20260724-howuipageswork/builder.png)

A **Widget** is any visual element that takes input from, or shows output to, the user. Widgets connect to other elements (most commonly a data source) and carry their own individual configuration.

![Widget](https://image.j-roque.com/posts/20260724-howuipageswork/widget.png)

 A **Task**, what used to be called a Data Source in earlier versions, is how a UI Page actually fetches data to feed its widgets. As of v11, Tasks are grouped into distinct categories: `Data Source`, `Transaction` (Business Workflows, service calls), `Core`, `KPI`, `Area`, `Material`, and `UI Page` (for publish/subscribe behavior between pages).

![Task](https://image.j-roque.com/posts/20260724-howuipageswork/task.png)

A **Property** belongs to the page itself and persists across saves — effectively a page-scoped variable used to initialize, receive, or store data that needs to move between elements. 

![Property](https://image.j-roque.com/posts/20260724-howuipageswork/properties.png)

An **Action Button** lives on the page's ribbon rather than the canvas, but is configured exactly like a Button widget. And a **Link** is the wiring that makes all of the above talk to each other: connect one component's output to another's input, and the page reacts.

![Action Button](https://image.j-roque.com/posts/20260724-howuipageswork/actionbutton.png)

## Converters: Bridging Type Mismatches

Links only work cleanly when the output type on one side matches the input type on the other. When they don't, for example a widget outputs a string, but the target expects a boolean, a **Converter** sits in between and reshapes the value.

![Converter](https://image.j-roque.com/posts/20260724-howuipageswork/converter.png)

Version 11 organizes converters into categories, and the walkthrough of the documentation surfaced a handful worth knowing by name rather:

- **Transformation** — e.g. `AnyToAnyProperty`, which pulls a single named property off an object (grab a row's `Id` column, for instance), and `AnyToEmptyArray`.
- **Object Handling** — array length, filter value, map value — the converters you reach for constantly when shaping a payload before it hits a DEE.
- **Logical** — `IsEqual`, `IsFalse`, `IsNotNull`, and similar boolean operators, usable directly without writing an expression.
- **Entity** — most notably `LoadEntity`, which takes an ID and resolves the full entity.
- **Resource** and **Material** converters, and more specialized ones layered on top of those.

Later on we will use `AnyToAnyProperty` to real use twice.

## Layouts and Overriding Native UI Actions

A **Layout** governs how a page's disposition changes with screen size — how many columns the Builder shows, whether side panels are visible, and how they're arranged. New layouts aren't hardcoded; they're added as entries in the native `UIPageLayouts` smart table, where you set a description and the column configuration to display in the Builder.

Native UI Actions, the built-in views the product ships, like the Fab Explorer's step view, can be overridden the same declarative way, through entries in the `UIPageContext` smart table, keyed by the action's identifier.

Create a custom page, add an entry in `UIPageContext` pointing the native step-view action at it, set the custom page as effective, and refreshing Fab Explorer now renders the custom page instead of the product default. Worth flagging: not every native UI Action can be fully replaced this way, some resist a complete override.

![Override Product Page](https://image.j-roque.com/posts/20260724-howuipageswork/override_product_page.gif)

## Presets: One Page, Many Contexts

**UI Page Presets** solve a specific, recurring problem: you want the same page reused across many contexts, each with different data, without building a dedicated page per context. The classic case is a dashboard button injected per-resource, same page design, different resource passed through as context depending on which button was clicked.

In the demo we will build exactly that: a preset scoped to `Menu Entry`, targeting the Manufacturing page, configured to inject one quick-access button per resource. Each button, when clicked, passes that resource's identity into the page as context; a `GetObjectByName` service-call Task resolves the full entity from that name, and the result feeds an Entity Details widget. Critically, each injected button can be scoped independently by role — in the demo, one resource's button was restricted to a `Cookie Manager` role while another was left open to all users.

The advantage over the old approach is real: previously, supporting this pattern meant a dedicated page per context, built and wired by hand. With presets, you build the page once, and the preset configuration decides how many buttons appear and who can see each one, no additional customization per resource.

![Presets](https://image.j-roque.com/posts/20260724-howuipageswork/presets.png)

## Anatomy of a Real UI Page: Building a Two-Level Sliding Grid

Let's imagine the user wants have a list of filters (container, facility, state) and a list view with two levels: Container View and Material By Slot Number. The user must be able to view the container detail info in the right side (Name; Orientation, Used Positions). We will add a sliding grid to show, green if the percentage is above 50% and red if it's less.

### Level 0: filters, a query, and clean naming

The first widget dropped onto the canvas was a Filter widget — deliberately renamed to `ContainerFilterWidget` rather than left at its default `FilterWidget1234`. That naming discipline isn't cosmetic; it's what keeps a page's link graph legible once it has more than a handful of widgets on it.

The Sliding Grid itself was configured in lazy ("on demand") mode, with two levels defined and single-selection mode enabled — a direct consequence of the requirement that selecting a container has to populate the right panel. A Query task, `GetContainerInfo`, supplied the first level's columns (name, orientation, user position, total position), and the Filter widget's output was linked straight into the query's filter input, no converter needed, since both sides were already the same object type.

> When creating task query make sure you select the appropriate retrieval time (Retrieve Data on Start/ Retrieve Data on Changes), as this impacts the responsiveness of the page.

![Build level 0](https://image.j-roque.com/posts/20260724-howuipageswork/build_UI_0.gif)

![Adding Filters](https://image.j-roque.com/posts/20260724-howuipageswork/adding_filters.gif)

### Level 1: a DEE as a data source, and the converter that fixes it

The second level deliberately used a different kind of Task, a **DEE Action** instead of a query, specifically to show both paths side by side. The DEE, already written beforehand, took a container ID as input and returned a data table of the materials, products, flows, and steps associated with it.

Wiring it up surfaced two real, live bugs, the kind every implementation runs into at least once.

**First bug:** clicking a level-0 row triggered the level-1 fetch, but the DEE failed immediately. Opening the browser dev tools showed why: instead of receiving the container's ID, the DEE was receiving the *entire row object* from the sliding grid's "fetch level" event. The fix was a converter — `AnyToAnyProperty` — inserted on that link, extracting just the row's `Id` field, then a `SetMapValue` converter injecting it into the DEE's input as a new key, `ContainerId`.

**Second bug:** with the ID now flowing correctly, the DEE executed successfully — visibly, in the dev tools' network response — but the grid still rendered no rows. The mismatch here was shape, not identity: the DEE returned a dictionary, while the Sliding Grid expected an array of a specific type. Same converter category solved it — another `AnyToAnyProperty`, this time pulling the `Result` field out of the dictionary into the shape the grid needed.

Neither bug was exotic. Both are the direct, unglamorous consequence of connecting two components that don't share a type — which is the entire reason Converters exist as a first-class concept in the Builder rather than an escape hatch.

![Build level 1](https://image.j-roque.com/posts/20260724-howuipageswork/level1_UI.gif)

### The right panel: entity details wired to selection

The right-hand panel used an Entity Details widget bound to a Property — `Container`, typed as a reference to the Container entity. Selecting a row on the Sliding Grid fires a selection-changed event, which writes the selected container into that property; the property, in turn, pushes straight through to the Entity Details widget. No query, no DEE, no round-trip, the selected row *is* the entity, just handed off through a property acting as a pipe.

![Right Panel](https://image.j-roque.com/posts/20260724-howuipageswork/level2_rightpanel.gif)

### The pagination bug hiding behind a page that "looked done"

At this point the page rendered correctly, filtered correctly, drilled down correctly, and populated the right panel correctly. But it can have a serious problem: the grid's pagination was happening entirely client-side.

That means on first load, the query fetches *every matching row*, and the browser slices it into pages after the fact. With ten rows in a demo, that's invisible. With three thousand, five thousand, or ten thousand containers, which is the actual scale most of these grids run at in production, that's a page load that silently pulls the entire table across the wire before showing you the first ten rows.

> A grid that "just works" against ten demo rows can still be pulling the entire table on every load — the browser hides that cost by paginating locally, right up until someone opens it against real production data.

The fix: add two integer properties to the page, `PageNumber` and `PageSize`. Wire the Sliding Grid's page-change and page-size-change events into those properties, and wire those same properties *forward* into the query as filter parameters, so the query itself only ever asks the database for the page currently being viewed. The query's total-record count feeds back into the grid so it can render the correct number of pages. Get this chain right, and the database only ever returns the rows actually on screen; get it wrong, and the grid is a convenient UI sitting on top of an unbounded query.

![Server Side Loading](https://image.j-roque.com/posts/20260724-howuipageswork/level2_serversideloading.gif)

### Template columns without hardcoded styles

The last piece of the requirement, a capacity column rendered as a green or red progress indicator using a custom column template with conditional syntax referencing the row's data (`user positions` over `total positions`, colored green above 50%, red at or below it). The one deliberate choice worth calling out: the demo used the product's existing CSS classes and CSS variables for the coloring, rather than hardcoded style values. Hardcoded styles silently diverge the moment the page renders under a different visual theme; variables inherit whatever theme is active automatically.

![Custom Column](https://image.j-roque.com/posts/20260724-howuipageswork/level2_customcolumn.gif)

```html
# if (data.UsedPositions != undefined && data.TotalPositions != undefined) { #
    <div>
        <div style="display: flex; flex-direction: row;width:100%;height:100%;align-items: center">
            <div class="progress" style="position: relative;width:100%;margin-bottom:0px;background-color:transparent;">
                <div class="progress-bar" role="progressbar" aria-valuenow="#: ${ percentageProperty} #" aria-valuemin="0"
                aria-valuemax="100" style="background-color:#: (UsedPositions / TotalPositions > 0.5 ) ?
                "var(--color-green)" : "var(--color-dark-red)" # ;position: absolute;width:#: UsedPositions #%">
                </div>
                <div style="position: absolute;width: 100%;color:var(--styleColor004);font-weight: bold;"
                align="center">#: UsedPositions #/#: TotalPositions #</div>
            </div>
        </div>
    </div>
# } #
```

## Business Workflows: The Low-Code Way to Build a DEE

A Business Workflow can be standalone, or bound to an operation event (Track In `Pre`/`Post`, Track Out, Move Next — the native operations MES already exposes). You define its inputs — Material, Resource, whatever the scenario calls for — and drag in prebuilt building blocks like *Dispatch and Track In*, wiring each block's required fields back to the workflow's custom inputs. Save it, and the product automatically generates the DEE underneath — test condition, action code, the whole apparatus — without a single line of C# written by hand. Wire an Action Button on a UI Page to trigger it, feed it Material and Resource straight from page properties, and clicking the button runs the generated DEE exactly as if someone had written it.

Also, these low-code building blocks aren't just an internal shortcut, they can be packaged and delivered to customers directly, letting their own teams compose new behavior out of pre-built, tested "Lego pieces" without opening a customization request at all. That's the same extensibility philosophy behind DEEs in the first place, just one layer higher and without the code.

![Button Biz Workflows](https://image.j-roque.com/posts/20260724-howuipageswork/button_bizworkflows.gif)

## Best Practices

The closing list reads like a checklist, but every item traces back to something that actually broke on a real project:

- **Avoid redundant entity loads.** Reloading an entity through a converter when a dedicated Property already holds it triggers unnecessary, repeated loads every time an action runs.
- **Eliminate duplicate progress indicators.** Multiple spinners on one page read as broken to an operator, even when nothing is actually wrong. Centralize on one.
- **Use server-side pagination, not client-side** — the exact bug the demo walked into. 
- **Run data sources only when needed** — evaluate per Task whether it should trigger on load, on change, or both; triggering more than necessary directly slows the page down.
- **Use CSS variables, not hardcoded grid styles**, so template columns stay correct across different visual themes.
- **Always measure performance against realistic data volumes**, not the handful of rows a demo happens to use.
- **Follow a naming convention for widgets and links.** `ContainerFilterWidget` beats `FilterWidget1234` the moment a page has more than five links to untangle.

## Final Thoughts

What separates a UI Page that survives a real deployment from one that gets rebuilt six months later isn't the widgets chosen. It's whether pagination was wired all the way through to the query, whether the output stayed properly typed at every converter boundary, and whether "it works in the demo" was ever tested against something closer to the size of the actual database.

The Builder will let you skip every one of those steps and still show you a page that looks finished. Production data is what actually checks your work.

> This blog post was based on a talk by José Novo in 2025-09-11 @CM-Portugal
