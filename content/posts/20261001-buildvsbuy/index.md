---
title: "Build vs Buy in the Age of AI"
description: "AI made building cheaper. It did nothing to the reasons companies stop building."
summary: "AI collapsed the cost of writing software, not the cost of owning it. The reasons manufacturers end up buying are the same as they were a decade ago, and AI adds a new one."
categories: ["Opinion"]
tags: ["AI", "MES", "Digitalization"]
date: 2026-10-01
showInRSS: false
draft: false
authors:
  - Roque
---

AI did not invent the build vs buy debate. It made building cheaper and owning more expensive, and most of the noise right now comes from people who only priced the first part.

## The Weekend Rebuild Myth

Every week there is a new post announcing that AI killed SaaS. Companies no longer need to buy software, the argument goes, because an agent can write it for them over a weekend. Uli Behringer made the most visible version of this case recently, arguing that SaaS should die and give way to AI-native systems.

Francisco Almada Lobo took the economics of that claim apart in [Killing SaaS won't democratize software](https://www.linkedin.com/pulse/killing-saas-wont-democratize-software-francisco-almada-lobo-qdixe/). His core point: you don't escape dependency by rebuilding. You swap a software vendor for a handful of foundation model providers, in a market with even higher barriers to entry.

I want to look at the other side, the one I see from inside implementation projects. Why do companies that build end up buying anyway, and why does AI leave those reasons intact?

A disclosure before going further: I work at Critical Manufacturing, an MES vendor, and Francisco is our CEO. Weigh what follows accordingly. I'll try to earn it anyway.

## This Debate Is Decades Old

Medium and large manufacturers have always had the option to build. They have IT departments, they have engineers, and many of them have tried. Semiconductor fabs ran homegrown MES for years before commercial products could handle their complexity.

The barrier to entry for software was never the problem. A competent team could always write a work order screen, a dashboard or a traceability table or spring up an MQTT broker with an UNS structure. What AI changed is the speed of producing that first version. It did not change what happens in year two, year five and year ten.

## Two Kinds of Pressure

To talk about building, you have to separate who is doing it. Organizations in a digitalization journey face pressure from two directions.

### Bottom-up: the industrious engineer

The first, and the most often forgotten, is bottom-up. People are ingenious. A process engineer is tired of copying values from an HMI into a spreadsheet, so they write a script. A quality technician builds a small app to track a recurring defect. A shift lead builds a dashboard nobody asked for and everyone ends up using.

These people were building long before AI, with Excel macros, Access databases and Python scripts. Their problems are small, specific and bound to how their organization works, and their budgets are close to zero. That is why they have always been hard to sell to. They create all those small ecosystem applications that are so resilient to change and are a bottleneck to most centralized digitalization projects, because they solve their narrow problem really well. Even when their root cause for existing no longer exists.

### Top-down: the centralized mandate

The second is top-down, pressured by the market. A CEO or CTO looks at the business landscape and decides the company needs a centralized approach. They need operational efficiency across sites, and above all they need accurate information on what is actually happening on the shopfloor, not what the weekly report says is happening.

This is the problem [enterprise software like an MES](https://j-roque.com/posts/20251209-whatisanmes/) exists to solve, and it is where the build vs buy decision lives. In my experience, the pattern has been consistent: build, struggle, buy.

> Bottom-up builders solve problems. Top-down mandates need standardization and centralized systems.

For decades these two pressures lived in separate worlds. The script on the engineer's laptop never threatened the plant-wide system. AI is what is about to change that, and I'll come back to it.

## Why Building Fails Slowly

Most in-house systems don't fail at go-live. The screens work, the data flows, the requirements are met. The failure is slower: the requirement itself moves, and the system can't move with it. There are four reasons this happens, and AI fixes none of them.

### You are buying a trajectory, not a feature set

When you buy a product, you are not buying today's features. You are buying into a multi-year journey of releases, regulatory updates, security patches and capabilities you have not asked for yet. Francisco puts it well: the value of enterprise software lies in continuous evolution, compliance updates, domain knowledge from implementations, reliability, and accountability, not in the code itself.

A product built for many customers is, by necessity, more general than anything you would build for yourself. That generality is the point. It is the room that lets the software follow you when your business changes. In MES, this is why the [data model is configurable rather than hardcoded](https://j-roque.com/posts/20260728-mes-dynamic-model/), and why [extensibility](https://j-roque.com/posts/20250725-iot-extensibility-i/) is a design concern from day one instead of a patch applied later.

A system built to your exact shape fits perfectly, right up until your shape changes.

### You judge your problems well, and your solutions badly

Organizations are reasonably good judges of their problems. They are poor judges of their solutions.

Fit for purpose is great if the purpose makes sense. The most common failure I see in digitalization is a company taking the processes that lived on paper and putting them on a screen. Same steps, same approvals, same workarounds, now with a login. It costs millions and changes nothing measurable.

AI makes this failure mode faster and cheaper to reach. Ask an agent to build what you describe, and it will build exactly what you describe, including every inefficiency you stopped noticing years ago.

### You cannot build for what you don't know

Understanding how other organizations solve a problem is how you stop solving it badly yourself. A product that runs across industries carries years of accumulated know-how from all of them.

I have watched this happen first-hand. We built a feature set for electronics manufacturing that ended up closing a key gap in our offer for medical device work cell lines. Nobody designed it for that. When it reached the people who knew that segment, it just clicked. No internal team at a medical device company would have built it, because nobody there had seen the electronics problem that produced it.

You can always build for what you know. You cannot build for what you have never seen.

### Infinite features, no one in charge

This is the reason that is new with AI, and it is where the bottom-up world finally collides with the top-down one.

When building a feature costs almost nothing, the harmless local script turns into an ungoverned shadow platform. Every team, every site and every engineer can now ship their own version of the solution. Decisions about how the plant runs get pushed down to local actors optimizing intensely for what they can see. Every one of those features still has to be maintained, secured, validated and integrated long after the person who prompted it has moved on.

The early data points the same way. GitClear analyzed 211 million changed lines written between 2020 and 2024 and found that [refactoring fell from 25% of changed lines in 2021 to under 10% in 2024, while copy/pasted lines rose from 8.3% to 12.3%](https://www.gitclear.com/ai_assistant_code_quality_2025_research). Google's 2024 DORA report estimated that a 25% increase in AI adoption was [accompanied by a 7.2% reduction in delivery stability and a 1.5% decrease in throughput](https://cloud.google.com/blog/products/devops-sre/announcing-the-2024-dora-report). The 2025 edition put it bluntly: ["AI doesn't fix a team; it amplifies what's already there."](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) Drop AI into an organization with no governance over its software, add stakeholders with completely different motivations and skill sets, and you have built a time bomb.

> AI made writing code nearly free. Nobody has made owning it free.

## Where the Critics Are Right

None of this means vendors are innocent. Per-seat pricing punishes growth. Lock-in is real. Shelfware is real: companies pay for modules they never switch on. And vendors bloat too; saying no is a discipline, not a guarantee.

Every one of those complaints is an argument for demanding much more from your vendors. More features, less issues, faster delivery, more innovation, all of these are reasonable demands. The issue of am I buying a partner with a future vision has never made so much sense. It's no doubt true that legacy big names who have for years being held by their sheer size and market, now face a bigger scrutiny and much more pressure from the market innovators.

## Where Building Makes Sense

So build, but build in the right place.

The key question is: what should you own because it differentiates you, and what should you consume because someone else operates it better? Your proprietary process know-how, your specific integrations, the logic that makes your plant different from your competitor's: build that, and use AI to do it faster. That is what extensibility in a platform is for, whether through [DEEs](https://j-roque.com/posts/20260724-howdodeeswork/) or a [code task in Connect IoT](https://j-roque.com/posts/20260922-csharp-codetask/).

Rebuilding work order management, genealogy or electronic batch records from scratch is not differentiation. It is paying to relearn lessons the rest of the industry already learned. But pushing a system to deliver the most value for you, that is a no-brainer. It's very hard to create value in an unconstrained foundationless system. Value is only accelerated on a foundation that enforces constraints and allows you to grow in the right direction.

## Final Thoughts

AI changed the cost of writing software. It did not change the cost of owning it, the value of what others have learned, or the difficulty of seeing past your own processes. Those were always the real reasons companies bought, and AI added one more.

Build what makes you different. Buy what makes you the same, and spend the difference on getting ahead.
