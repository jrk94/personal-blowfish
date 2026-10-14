---
title: "Make-to-Order in the Age of AI"
description: "Make-to-order never delivered on its promise of autonomy. AI agents running on top of an MES might finally get us there."
summary: "Make-to-order was always more about offloading risk than about autonomy. AI agents on top of an MES are the missing link: not a factory that makes anything, but one that reconfigures itself for any order within its boundaries."
categories: ["Opinion"]
tags: ["AI", "MES", "Digitalization"]
date: 2026-10-13
showInRSS: false
draft: false
authors:
  - Roque
---

Before AI sucked the oxygen out of every conversation, we had a different dream for the factory floor: a plant that reconfigures itself for any order. 

But the real promise was never the machines. It was the configuration labour: the engineering work of turning each custom request into a validated recipe, routing and quality plan. AI Agents can take much of that work over, as long as a system of record enforces what they are allowed to do.

*Disclosure: I work at Critical Manufacturing, which builds an MES. That shapes where I look for answers, so weigh the argument accordingly.*

## The Dream

We had already moved past lean manufacturing and kaizen into something more flexible and dynamic. The promise was simple: a customer places an order, and the factory fits itself to it. That is **mass customization**, in the sense Joseph Pine gave the term in his 1993 book *Mass Customization: The New Frontier in Business Competition*, and it is why consumer products that let you pick your own configuration, shoes being the usual example, became so visible.

The dream was never "a machine that makes anything". It was a factory where the cost of switching to a custom order approaches zero. Throughout this post, "within its boundaries" means exactly this: the set of products the factory's installed machines, tooling, validated process ranges and stocked or obtainable materials can physically and legally produce. Anything outside that set is a different conversation.

## Three Things Wearing the Same Name

The term "make-to-order" covers too much ground. Three different things hide under it.

**Make-to-order as a production strategy** is old and healthy. You build after the order arrives instead of building to forecast. Shipyards, capital equipment and configurable industrial products work this way today. Nothing is broken here.

**Contract manufacturing** is a business model. A customer outsources production to a supplier. Semiconductor foundries are the textbook case. The order sizes are large, and the customer and supplier are often deeply intertwined: the customer has to know the supplier's processes and machines in detail, and sometimes even buys the equipment the supplier will use. That is a smart way to share risk and skill set. It is a different thing from the dream.

**Autonomous reconfiguration for custom orders** is the real subject of this post. The factory takes a customer-specific request, and turns it into recipes, material flow, quality gates and machine setup, without a multi-week engineering effort per order. This one never arrived at scale.

> The promise was never outsourcing. It was making the cost of reconfiguring for a custom order near zero.

## Why We Never Got There

If the strategy exists and contract manufacturing works, the missing piece is the cost of reconfiguration. It breaks down into four bottlenecks. They are not equally solvable by software.

### Recipe and routing engineering

Someone has to translate "this customer wants this variant" into a recipe, a routing, parameters and quality checks. Today that is human engineering work, and my argument is that its effort grows with the number of variants you offer far more than with the number of units you build. Ten thousand identical units need one recipe. Ten thousand distinct variants need ten thousand decisions, even if most are small.

I am arguing this from structure, not from a measurement.

In [Critical Manufacturing MES](https://help.criticalmanufacturing.com/) you can already model products, flows and recipes and run make-to-order on top of them, but customer and maker still have to agree up front on the model and its configurations. Spreadsheets and third-party interfaces have tried to digitalize this interface, but the gaps is still there.

This is the bottleneck an agent can remove. It is a translation problem: structured knowledge about what the factory can do, mapped to a request. Language models are good at every kind of translation.

The capabilities side is not new either. Salvador, Martin de Holan and Piller, in ["Cracking the Code of Mass Customization"](https://www.researchgate.net/publication/265498057_Cracking_the_Code_of_Mass_Customization) (MIT Sloan Management Review, 2009), described mass customization as a matter of organizational capabilities: knowing where customer needs diverge, recombining existing resources, and helping customers specify what they need. The agent model maps onto the second and third almost directly.

### Qualification and validation

In regulated or high-reliability industries, a new recipe is not trusted until someone has qualified it. An agent can draft the recipe in seconds. It cannot make a regulator, or a quality department, comfortable with a recipe nobody has validated.

An agent can remove much of the effort: assembling evidence, checking a new variant against validated parameter ranges, preparing the documentation. The requirement itself stays, but the unit of qualification can change. Qualify the envelope instead of the individual order: validate the ranges once, and let agents work freely inside them.

That is also where the vision grows. Every validated range, building block and recorded outcome widens the envelope. A factory that learns from every order it executes, and from the evidence the MES already captures, can qualify its envelope faster each year. The envelope is not fixed. It is the thing the whole system gets better at expanding.

### Changeover and tooling

Changeover is physical: someone has to swap a die, load a different fixture or clean a line. A software agent cannot do that on its own, but a robot can. Robots and automated equipment already handle tool changes, fixture loading and material handling in many plants, and the agent's role is to orchestrate them: schedule the changeover, sequence orders to minimize it, dispatch the robot or the operator, and verify the result before production resumes.

Seen that way, changeover is not a wall. It is a capability gap that robotics is closing. Pair agents with robots and the loop closes: the agent decides what the next order needs, the robot sets the line up for it, and the MES confirms the setup is correct before releasing the order. If the factory is expected to switch between orders constantly, quick-change tooling, modular lines and automated setup stop being nice-to-haves and become the design brief for new equipment.

Today the economics decide how far this goes. Where changeovers still depend on manual work, agents shrink the cost at the margins with better sequencing and fewer surprises. Where robots take over the setup, the cost of a batch of one starts to fall for real.

Additive manufacturing attacks the same bottleneck from the other side: instead of changing tooling faster, it removes the tooling. That is why it was supposed to shortcut all of this. Unfortunately, material qualification, quality consistency, speed and scale still limit many production-grade uses today. As those limits move, it becomes one more machine in the capability model, with its own envelope, that the agents can plan for and the robots can feed. It is a piece of the answer, not a replacement for it.

### Material lead times

If a variant needs a component that is not on the shelf, no agent conjures it. An agent can detect the need earlier, request material in sync with the order and propose alternatives from what is already stocked. Today, the lead time of the supplier is still the lead time of the supplier. The longer-term picture is a supply chain whose systems speak to each other as fluently as the factory's own: the order's agents negotiating availability and alternatives with the supplier's systems directly, instead of a buyer chasing emails. That is not here yet, but it is the natural extension of the same idea.

### What that leaves

None of the four is a wall. Agents can remove much of the first and help with the second right away. Robots and automated setup let the same agents close the loop on the third, turning changeover from a manual job into something the system orders and verifies itself. The fourth waits on supply chains that speak to each other as fluently as the factory does. That is the shape of the vision: software that decides, robots that execute, and an MES that checks every step.

It is enough to start, because the first bottleneck is the one that, scales worst with variety. The others are engineering problems with a clear direction, and each one that falls widens what the factory can say yes to.

## AI as the Missing Link

My vision is an intelligent factory run by a set of agents that sit on top of an industrial operations platform. The system keeps the control, traceability and quality. The agents adapt the factory at the speed of software instead of the speed of an engineering change request.

Why the MES as the backbone layer, and not the ERP or the planning system? Because the MES is where the plan meets the shop floor: it is the system that knows which lot is on which machine, which recipe version was released, and whether an operation was actually performed. ERP and APS plan against abstractions of that reality. A model describes. It does not refuse to release an order. The MES does.

Let me walk one order through it. This is an illustration, not a description of a deployed system.

Imagine a plant that builds circuit boards for cars. A customer orders 200 interior lighting boards: the same board the plant already makes, but with amber LEDs instead of white, a different connector, and a protective coating, because this version will sit inside a door panel where moisture gets in. The customer brings a design and a requirement, not a process.

1. **Feasibility agent.** It receives the request and compares it with a structured model of what the plant can do: board sizes the line handles, components the placement machines can pick, coatings the plant applies, and the tests it can run. It uses the product and capability data in the MES. It answers one question: is this inside the envelope? If the customer asked for a connector too large for the placement machines, or a coating the plant does not apply, the request stops here, or comes back with the nearest feasible option.
2. **Planning and configuration agent.** For a feasible order, it drafts the route the boards will take (print solder paste, place components, solder them in the oven, coat, test, load firmware) and the recipes for each step from existing validated building blocks: the placement program with the new LEDs and connector, the oven temperature profile, the coating recipe, the test program. It uses the approved process parameters and ranges stored in the MES.
3. **Material and execution agents.** They check that the amber LEDs, connectors and coating are in stock, order what is missing, group the order with others that use the same machine setup, and prepare the settings for each machine.
4. **Quality agent.** It proposes the checks: a camera inspection of every solder joint, an electrical test of each board, an inspection of the coating. It verifies the results at each step against the specification.

### What the LLM adds over a configurator

Fair objection: this looks like a product configurator plus a rules engine. It is not, and the difference is the whole point of this post.

A product configurator configures the product. The customer picks amber LEDs, a different connector and a coating, and the configurator produces a valid bill of options. That is where it stops. Someone still has to turn that bill of options into a new placement program, an oven profile, a coating recipe, a test program, a material plan and a quality plan. That hand-off to engineering is exactly the gap make-to-order never closed.

Here, when the customer configures a product, behind the scenes, they are configuring the factory. Every choice they make ripples into routes, recipes, machine settings and inspections, and the agents carry that change through to the shop floor, with the MES checking each step. The customer never sees it. The factory adapts to their product, not the other way around.

Rules still have a place. If the options are a finite, fully enumerated catalogue and every combination has been engineered in advance, deterministic configure-to-order logic is cheaper, faster and easier to audit than any model. Use rules there.

The language model earns its place where the request, or the factory change it implies, does not fit a dropdown:

- **Free-text and unstructured requests.** "Same as the lighting board from March, but it now goes inside the door, so it has to survive humidity" arrives as an email, not as a form.
- **Near-envelope specs.** The request sits at or just beyond the edge of what was modelled, and someone has to judge which nearby feasible option actually serves the customer's intent.
- **Ambiguous intent into catalogue terms.** Translating what the customer means into the vocabulary of the factory's capability model.
- **Drafting candidate plans.** Proposing a recipe and routing from validated building blocks for a combination nobody has configured before, as input to validation, not as a decision.

Where every variant and its factory setup were engineered in advance, rules suffice and the agent adds risk without adding value. The case for agents grows as the degrees of freedom increase and get fuzzier and less enumerable, because that is where nobody can pre-engineer the factory for every answer.

### Where the guardrails bite

The MES is the referee. If the planning agent proposes an oven temperature outside the validated profile, the MES refuses to release the order. If the LED reel comes from a supplier not approved for automotive parts, the MES does not let production start. If an operation is skipped, tracking does not allow the unit to move on.

### Where a human still approves

Humans stay in the loop wherever the cost of being wrong is high or the decision is outside the validated envelope. In this example: approving any recipe that uses a parameter combination not seen before, signing off a first-of-its-kind variant, and handling exceptions the feasibility agent flags as borderline. The goal is not zero humans. It is humans reviewing exceptions instead of authoring every configuration.

### When an agent is wrong

Assume it will be. The planning agent picks a coating that is approved, but too thin for a board that will live inside a wet door panel. The guardrails catch the invalid, not the merely bad. That is why the quality agent verifies results at each step, why every decision and its inputs need to be recorded in the same traceability the MES already keeps, and why a human reviews the outcomes. A wrong configuration should cost you a held batch of boards, not a recall of cars. Designing for cheap failure matters more than designing for no failure.

### What must already be physically there

Agents do not change the physical world by themselves. For the lighting board example to work, the solder paste stencil for the board must exist, the placement machines must be able to handle the new components, the coating equipment and test fixtures must be in place, the LEDs, connectors and coating must be stocked or quickly obtainable, and either robots or operators must be available for the changeovers. The agent reconfigures the factory's logic, parameters and schedule, and directs the robots that do the physical setup. The tooling and the materials are the boundary.

## Boundaries Are the Point

A factory that makes anything you ask is unrealistic for the foreseeable future. A factory that adapts to customer demands within the boundaries of what it can make is a realistic target for the near term, and the first pieces exist today. A circuit board plant won't build a lighting board in one order and stitch a running shoe in the next. It will build every combination of board, component and coating that its machines and shop floor processes make possible.

Who owns the recipe? In this model, the factory's agents propose and the factory's validated envelope constrains them. A customer who brings their own proprietary process is the contract manufacturing case from earlier, and it needs its own agreement about who sees what. The dream of a factory that never needs to know the recipes of its customers is a separate problem, and I am not claiming agents solve it.

## Why This Might Not Work

Four objections are worth taking seriously.

**LLM reliability and auditability.** Language models make mistakes and cannot always explain them. The answer is architectural: agents propose, the MES enforces, and every decision is logged with its inputs. You do not need to trust the agent. You need to bound it and be able to reconstruct what it did.

**Validation and regulatory burden.** A recipe an agent wrote on Tuesday still has to be defensible on Wednesday. This limits the vision in regulated industries to orders inside a pre-qualified envelope. Outside it, the process falls back to the slower path. The gain is smaller there, not zero.

**Changeover economics for small batches.** If every custom order costs a long physical changeover, no agent makes a batch of one profitable. This is the strongest objection. The vision works where changeovers are already fast or can be sequenced across orders, and it does not where they are not. That is a reason to pick the first use cases carefully.

None of these kill the idea. They define where it starts.

## Final Thoughts

Make-to-order stalled less because of the strategy and more because turning each custom request into a configured, validated factory was human work that become very hard to maintain and wouldn't scale. AI agents on top of an MES can take much of that translation, within guardrails and with humans on exceptions. They will not change tooling or shorten supplier lead times.

If you want to test this, start where changeovers are already fast and requests arrive as messy free text, such as quoting and drafting routings for near-standard variants, with the MES validating every output before release.

The factory won't make anything. What changes is the price of asking it for something different.
