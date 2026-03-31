---
title: "Why UNS is not for Semiconductors?"
description: "Why UNS is not for Semiconductors?"
summary: "Why UNS is not for Semiconductors?"
categories: ["Opinion"]
tags: ["UNS", "Thoughts"]
date: 2025-12-11
draft: false
authors:
  - Roque
---

# Why the Semiconductor Industry Doesn’t Need a Unified Namespace

## Introduction

Unified Namespace was a great innovation, not of technology but of architecture and philosophy. In industrial automation, the **Unified Namespace (UNS)** as a concept is gaining not just traction as a solution to fragmented data systems, but a way to pivot traditional manufacturing, making industry 4.0 a more tangible and reachable goal. It aims to centralize operational data in real-time, enabling machines, sensors, control systems, and business platforms to speak a common language using standards like MQTT. While this is transformative for industries that are behind on their automation and data journey, with siloed systems and rigid ISA-95 hierarchies, it **misses the mark in the semiconductor sector** [22]. In the last MESI 4.0 Summit 2025, I had an opportunity to watch live [Walker Reynolds of 4.0 Solutions](https://www.youtube.com/@4.0Solutions), pitching all of the virtues of UNS and its ability to impact industries.

{{< youtube _bZwzm663fM >}}

Semiconductor manufacturing is already one of the **most automated and digitally integrated industries** in the world. Leading-edge fabs operate under tight cleanliness, precision, and yield constraints that have driven extreme automation and deep system interoperability for decades. The goal for decades has not just been to collect data, it became about a deep control of everything that happens in the shopfloor in real-time. In this environment, implementing a UNS adds **redundancy, not value** [23].

---

## Why UNS Works for Lagging Industries

In many sectors like food processing, packaging, or metalworking, **data fragmentation is rampant**. Machines often run in isolation, in the best case, with their data passing through siloed layers: PLCs to SCADA, then to MES, and maybe ERP. This pyramid structure delays insights and creates fragile, expensive integrations. UNS addresses this by creating a real-time, context-aware, centralized data layer where devices publish to a shared broker, dramatically reducing integration costs and latency.

UNS is therefore a **modern solution to decades-old problems**, ideal for low-tech or digitally immature operations. It democratizes access to OT data and opens paths to analytics, dashboards, and AI-driven optimization — **but only where these fundamentals are lacking**.

It doesn't try to be a complex centralized system of shopfloor control, but it leverages decentralized information silos in the shop-floor. It adds context hooks to information from data nodes through the factory. This enables with little effort to move from a babel, where everything talks different languages, into a Babel that comes with yellow pages, of where the information is coming and what it's about. This provides the end user with just enough to try and manipulate and aggregate the data to create information for real-time monitoring.

---

## Semiconductor Manufacturing: The Outlier

### The Most Automated Industry

Semiconductor manufacturing is built on **extreme levels of precision, cleanliness, and scale**, which makes manual intervention infeasible. In state-of-the-art 300mm wafer fabs, **over 85% of wafer transport is handled by robotics**, and **lights-out operations** (with no human intervention) are standard [1]. ASE alone operated **56 fully lights-out factories** by 2024 [2].

All new 300mm fabs now use **automated material handling systems (AMHS)** such as overhead hoist transport (OHT), which eliminates manual wafer movement. By 2024, **68% of fabs deployed OHT**, up from 52% in 2020, with the average number of OHT vehicles exceeding 470 per fab [3]. These systems handle more than **80% of all wafer movement** in advanced fabs [4], and help cut **human-induced defects by nearly 60%** [5].

Compared to other industries, the semiconductor sector is **decades ahead**. In 2020, semiconductors and electronics overtook automotive in global robot adoption, accounting for **29% of all new industrial robot installations**, versus 21% in automotive and ~3% in food and beverage [6][7].

{{< youtube h5I8i9hNcLU >}}

---

### Deep, Standardized Integration

While other sectors scramble to unify OT data, semiconductors already **solved that problem in the 1980s and '90s**. For semiconductors, the challenge becomes not how to incorporate these systems in the shopfloor, but how to update all the systems that were built in the past in new software that can give them platforms for advancing into the future. These systems, have become very complex, highly customized and tuned to their companies particular realities. As their systems become legacy, they require strategies to build programs to standardize and migrate into new platforms.

The industry created the **SECS/GEM communication standard** ([Secs-GEM blog post](https://j-roque.com/posts/20250227-secsgem/)), ensuring every tool can report its state, receive recipes, and be managed centrally [8]. Today, fabs use MES (Manufacturing Execution Systems) and host computers to orchestrate thousands of tools and steps with **nanometer-level accuracy and zero human touch** [9]. MES go to such a level of control, that the industry created the SEMI E142 standard for wafer mapping. The MES tracks everything that occurred in each layer of the die and is always interfacing with process machines to keep them informed of everything contained in the wafer [19]. 

![Wafer Map](https://image.j-roque.com/posts/20251211-uns-semiconductors/wafermap.jpg)

![Wafer Map E142](https://image.j-roque.com/posts/20251211-uns-semiconductors/wafermap_e142.png)

Machine Learning also is becoming ubiquitous, either with direct machine interfacing, collecting a whole set of machine events to perform accurate predictions, or with advanced algorithms for AI-powered image classification [20].

{{< youtube mj8iWMnJMpI >}}

This level of integration means a typical fab already has a **"practical UNS"** embedded through SECS/GEM protocols and centralized scheduling. Machines, MES, and even ERP are already linked in real-time. They are able not just to monitor but also to have direct real-time control of everything that happens in the shopfloor.

---

### Advanced Analytics Already Embedded

Fabs have long relied on **Statistical Process Control (SPC)** and **Advanced Process Control (APC)** to optimize operations. These have evolved into ML-powered **virtual metrology**, **predictive maintenance**, and **real-time fault detection** (FDC). For example, **Amkor increased engineering productivity by 60%** by deploying FDC systems [10].

TSMC pioneered **AI-driven visual inspection** and **automated wafer warehousing**, reducing manual handling by 95% [11]. A single fab can generate **millions of data points per day** [12], all flowing into tightly coupled analytics engines. McKinsey estimates that AI could add **$85–95 billion in annual EBITDA** to the industry by 2025 [13].

---

## Investment in Automation: A Decade of Acceleration

From 2020 to 2025, semiconductor firms poured **billions into automation and smart fab initiatives**. Global capex for semiconductor equipment reached **$114 billion in 2024**, and projections show over **$500 billion in fab investment by 2030**, much of it focused on automation infrastructure [14][15].

Additionally, **35–40% of equipment R&D budgets now target software development** (MES, control, analytics), up from 15% a decade ago [16]. Unplanned downtime can cost **over $1 million per hour**, further driving investment into AI, APC, and resilient automation [17].

---

## UNS in a Fab: Solving a Problem That Doesn't Exist

Implementing UNS in a fab would be:

- **Redundant**: Equipment is already connected to centralized control systems via standardized interfaces (SECS/GEM, EDA).
- **Risky**: Semiconductor fabs are uptime-sensitive and data-secure environments. Introducing a broad, generic MQTT layer could introduce vulnerabilities.
- **Oversimplified**: UNS abstracts data context in ways that might work in low-tech operations, but **fab data is deeply contextual** (e.g., recipe ID, slot mapping, process window).
- **Disruptive**: Fabs already have orchestration frameworks, such as factory-level automation controllers, tailored to coordinate MES, AMHS, and dispatch logic in real time [18].

---

## In Contrast: Where UNS Makes Sense

UNS works well for sectors that:

- Still rely on paper or spreadsheet-based production tracking.
- Lack standardized machine interfaces.
- Have siloed OT and IT networks.
- Are beginning their Industry 4.0 journey.
- Very Mixed industry companies with different levels of maturity.

In these settings, UNS can accelerate modernization, enable analytics, and reduce costly system integration efforts. But that’s **not the case in semiconductors**.

---

## UNS Is for Catch-Up, Not for Leaders

The semiconductor industry is not a typical industrial environment. It’s a **mature, automated, data-rich sector** with decades of experience integrating machines, software, and data. It has already **solved the problems** UNS was invented to address.

While UNS can help industries with fragmented data catch up to the digital age, **semiconductor fabs are already there**. Adopting a Unified Namespace in this context is like installing a garden hose next to a high-pressure water main: unnecessary and potentially disruptive.

**Let low-tech factories unify their namespaces. Fabs already did – decades ago.**

---

## Still on the Fence?

Don't get me wrong, UNS has a lot going for it. The truth is, if you are at a high level of maturity, emitting UNS events for all your operations should be as simple as pushing a button. As we move downstream in the supply chain and leave the semi space. UNS has an ability to quickly create a scalable data monitoring system, avoiding having to deal with all the complexity of actual process control. With integrated supply chains, it has opened the use of ISA95 to be the lever on which we can tie in and understand information.

The question should not be, does your MES system support UNS. It must be does your MES support UNS and shop floor control and machine integration. UNS can be a great place to start, but it's about the first step in industry 4.0 not the last one. You need to have a platform able to support your journey, it's ok to start with UNS, but you need to choose a platform that allows you to grow and start moving to a real-time control of your shop-floor [21].

{{< externalcard
  title="Democratizing Manufacturing Analytics – From CDM to UNS"
  description="How UNS bridges analytics and real-time operations in smart manufacturing."
  image="https://image.j-roque.com/posts/20251211-uns-semiconductors/UNS-CM.png"
  url="https://www.criticalmanufacturing.com/blog/democratizing-manufacturing-analytics-series-2-from-cdm-to-uns-bridging-manufacturing-analytics-and-real-time-operations/"
>}}


[1]: https://standardbots.com/blog/lights-out-manufacturing  
[2]: https://www.aseglobal.com/csr/sustainability-governance/smart-factory  
[3]: https://www.marketgrowthreports.com/market-reports/semiconductor-oht-overhead-hoist-transport-market-103785  
[4]: https://www.marketgrowthreports.com/market-reports/semiconductor-oht-overhead-hoist-transport-market-103785  
[5]: https://www.intelmarketresearch.com/robotsemiconductor-market-9793  
[6]: https://ifr.org/img/worldrobotics/Executive_Summary_WR_Industrial_Robots_2021.pdf  
[7]: https://www.weforum.org/press/2022/02/semiconductors-electronics-and-pharmaceuticals-lead-digital-transformation-in-manufacturing  
[8]: https://www.semi.org/en/standards-watch-2022-Sept/intro-to-semi-communication-standards  
[9]: https://thl.com/articles/thls-podcast-ep-3-automation-in-action-beyond-the-chips-how-automation-supercharges-semiconductor-production  
[10]: https://semiengineering.com/smart-manufacturing-makes-gains-in-chip-industry  
[11]: https://www.yolegroup.com/industry-news/tsmc-pioneers-the-worlds-first-automated-wafer-inbound-outbound-system-reducing-13-million-manual-tasks-per-year  
[12]: https://www.semi.org/en/blogs/technology-and-trends/semiconductor-manufacturing-in-the-industry-40-era-an-ai-use-case  
[13]: https://www.semi.org/en/blogs/technology-and-trends/semiconductor-manufacturing-in-the-industry-40-era-an-ai-use-case  
[14]: https://www.intelmarketresearch.com/robotsemiconductor-market-9793  
[15]: https://www.intelmarketresearch.com/robotsemiconductor-market-9793  
[16]: https://www.questglobal.com/insights/thought-leadership/modernizing-semiconductor-equipment-development  
[17]: https://worktrek.com/blog/predictive-maintenance-trends  
[18]: https://www.criticalmanufacturing.com/blog/lights-out-automation-creating-resilient-factories  
[19]: https://www.criticalmanufacturing.com/wp-content/uploads/2022/06/CMF211863-Data-Sheet-Mapping.pdf  
[20]: https://www.criticalmanufacturing.com/mes-for-industry-4-0/apps/c-alice/
[21]: https://www.criticalmanufacturing.com/blog/democratizing-manufacturing-analytics-series-2-from-cdm-to-uns-bridging-manufacturing-analytics-and-real-time-operations/
[22]: https://www.symphonyai.com/industrial/unified-namespace-complete-guide/#:~:text=A%20Unified%20Namespace%20,point%20integrations
[23]: https://www.criticalmanufacturing.com/industries/semiconductor-manufacturing/