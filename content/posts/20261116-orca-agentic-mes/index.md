---
title: "Orca, Worktrees and the First Iteration of an Agentic MES Project"
description: "One Claude Code session per git worktree, a lifecycle from work item to draft PR, and two human gates. The first iteration of an agentic workflow for building MES customization projects."
summary: "How I use Orca to run one agent per worktree on Critical Manufacturing MES customization work: the lifecycle, the skills and agents behind each stage, a real walkthrough of the first story, and an honest list of what is still unproven."
categories: ["AI"]
tags: ["AI", "Agents", "MES", "Orca", "Claude Code"]
date: 2026-11-16
showInRSS: true
draft: false
authors:
  - Roque
---
An agent that writes code is a demo. An agent that picks up a work item, builds it in isolation, proves it, tests and screenshots it, gets reviewed by two other agents and stops at a human gate is the beginning of a process.

As models got better our way of work also changes. We stop interfacing with VSCode asking questions to a chatbox and start shifting towards a different role. The role of thinker, architect, business owner and critical thinker, but less and less that is about being the old translator from human requirements to computer programs.

This post is about that shift, and about starting. Everything below is iteration one. Of the 40 skills behind it, 11 are still drafts that route their work to a human, so the unattended path covers the code-centric artifacts (services, DEE actions, GUI, tests) and not yet most of the data, reporting and IoT families. I will show which.

*Disclosure: I work at Critical Manufacturing, which builds an MES. The project structure I lean on is ours, so weigh the argument accordingly. The `.claude/` folder this post describes is not public today. It may be shared later, so nothing here links to it.*

## The End Goal

There are two approaches to trying to do an agentic flow. You can go the puzzle approach, or you can go the skeleton approach. I am more of a skeleton guy. I build the main flow and then each of the missing skills or parts of the projects that are not as refined will be filled in as I work on stuff that touches them, meanwhile I strengthen the core ideas.

![puzzle skeleton](https://image.j-roque.com/posts/20261116-orca-agentic-mes/puzzle-skeleton.png)

The end goal is a fully agentic workflow to build a Critical Manufacturing project. In this post **fully agentic** means one thing: unattended between human gates. Agents do the build. Humans own the decisions and the acceptance, at gates defined up front. Not absent, placed.

I think this is tractable for one specific reason. An [MES project is a repository](https://j-roque.com/posts/20261014-cmfaudit/) of typed packages: `Data`, `Business`, `HTML`, `IoT`. Each requirement turns into an artifact, each artifact lives in a package type, each package type has a specialist, and each specialist has a Definition of Done. That is a catalog, and a catalog is something you can hand to agents.

Oddly enough this kind of loopsback to my argument on [Make-to-Order in the Age of AI](https://j-roque.com/posts/20261013-maketoorder/): agents propose, a system enforces, humans take the exceptions. Here the "system" is a toolchain, hooks and reviewers instead of an MES. Same shape.

## Why Orca

There is a total paradigm shift when moving from writing code, to managing and validating agents. Different requirements require different cockpits. I chose Orca as one of the many possibilities out there, I wanted it to be model provider agnostic and with multi harness support, with also a native worktree management support. It's true that Orca for me is not ideal as it has first class support for github but not for Azure DevOps, but still those felt like a nice to have and not a blocking point. Having MCPs for ADO, makes it less important for my ticketing platform to be integrated with my agent management platform.

[Orca](https://github.com/stablyai/orca) runs one Open Code/Codex/Claude Code session per git worktree and shows each workspace as working, waiting or done. A plain `git worktree` gives you the folder. Orca adds the parts this workflow leans on:

- **Per-worktree setup.** `orca.yaml` runs a setup script when a worktree is created and an archive script when it is removed. With `setupAgentStartupPolicy: wait-for-setup`, the agent does not start until the devcontainer, restore and stack seeding are done.
- **Tabs.** Each worktree opens an Agent tab, a Devcontainer shell and a Stack tab tailing the background stack log.
- **A status line.** The agent writes it with `orca worktree set`: "slice 2/4 green; running live tests". I read the state of a run from the card, not from a terminal.
- **A waiting state.** When a turn ends on a question, the workspace shows *waiting*. A blocked agent is visible instead of silently idle, which matters because the human is the scarce resource in this system. Orca also notifies; I have not leaned on the mobile side enough to claim more than that it exists.

**A worktree per work item** also gives isolation and parallelism: its own branch, folder, devcontainer and local MES stack. Two agents cannot trample each other's build output or database.

Orca is the cockpit. The process lives in a `.claude/` folder of skills, agents and hooks, kept **project-agnostic**: no customer, package or repo names. Every project fact (packages, types, versions, base branch, test projects) is read live through a `cmf-project-context` skill. Facts that go stale in prose do not go stale when you query them.

Most of our developers, partners and clients, work in several projects at the same time, therefore the solution could not be very bespoke to a particular project. Orca allows easy switching through multiple repos and to multiple work items in the same repo.

## The Lifecycle

Github or Azure DevOps (ADO) work item tags are the state machine: `agent:needs-refinement`, `agent:needs-info`, `agent:ready`, `agent:ready-for-human`, `agent:in-progress`, `agent:blocked`, `agent:in-review`. Exactly one at a time. Anyone can see where an item is without opening a terminal.

{{< mermaid >}}
flowchart LR
  WI[ADO User Story or Bug] --&gt; R["/refine<br/>human + Claude"]
  R --&gt;|brief approved<br/>agent:ready| O[Orca worktree<br/>id-slug]
  O --&gt; S[orca.yaml setup<br/>devcontainer, restore, stack]
  S --&gt; D["/deliver<br/>autonomous"]
  D --&gt;|draft PR<br/>agent:in-review| PR[Human PR review]
  PR --&gt;|comments, failed checks| I["/iterate-pr"]
  I --&gt; PR
  PR --&gt;|merge| Done[Work item resolved]
  Done --&gt; Retro[retro proposals<br/>reviewed PR into .claude]
  D -.-&gt;|blocked: question| H[Human answers in Orca or ADO]
  H -.-&gt; D
{{< /mermaid >}}


| Stage           | Who                         | What happens                                                                         | Main skills and agents                                                                             |
| :--------------- | :--------------------------- | :------------------------------------------------------------------------------------ | :-------------------------------------------------------------------------------------------------- |
| `/refine`       | Human + Claude (**gate 1**) | Turn a story or bug into an approved brief                                           | `refine`, `grilling`, `cmf-artifact-catalog`, `diagnosing-bugs`, `tdd-cmf`                         |
| Worktree        | Orca + script               | Branch `<id>-<slug>`, devcontainer, restore, own MES stack                           | `orca.yaml`, `cmf-worktree-env`                                                                    |
| `/deliver`      | Agents, unattended          | Slice loop, integrate, verify, review, ADO Test Cases, PR text, draft PR, debt check | `deliver`, specialists, `cmf-verifier`, `cmf-merger`, `cmf-reviewer`, `pr-description`, `cmf-debt` |
| PR review       | Human (**gate 2**)          | Read the draft PR, comment                                                           | none: the human reads what `/deliver` published                                                    |
| `/iterate-pr`   | Agents                      | Classify each comment as fix, answer or push back                                    | `iterate-pr`, specialists                                                                          |
| Merge and retro | Human + agents              | Merge; friction becomes proposals                                                    | `retro`, `cmf-skill-authoring`                                                                     |


### Stage 1: Refine, where the human works hardest

`/refine` is the one stage where I am in the loop the whole time, and it is deliberate. Every hour spent here saves several downstream, because an agent that has to guess will guess confidently. The counterargument is real: refinement has an upfront cost, and I have no measured payback for it yet. This came from a lot of what Matt Pocock's advocates and also with my experience as a developer getting user stories to work on.

Every part of the refinement is stored in the Work Item, every time there's a blocking point or an additional fork in the road the information is stored in the work item itself.

![Refinement History](https://image.j-roque.com/posts/20261116-orca-agentic-mes/refinement-history.png)

The output is an approved brief with Given/When/Then acceptance criteria, an artifact map, a Definition of Done, and the work cut into **tracer-bullet slices**: thin vertical cuts that each work end to end, published as child Tasks with predecessor links. The brief also carries a **Doubt Rule**: when `/deliver` hits ambiguity it cannot ask about, this rule says what to do, and the run records a deviation.

The classification step is where the catalog pays off. Each requirement maps to an artifact, a target package and a specialist through `cmf-artifact-catalog`, with a DoD matrix attached. A requirement that cannot be mapped is a finding, not a guess. A requirement that maps to a draft skill becomes `ready-for-human`.

### What a Slice Is

The slice is the unit everything else in the lifecycle is built around, so it is worth being precise about it.

A **slice** is one narrow but complete path through the system: the artifact, its test and its data, together, small enough that a fresh agent can finish it in one context and a verifier can prove it on its own. The term comes from the tracer-bullet idea: fire a thin round all the way through the stack first, so you learn early whether the path works, then widen.

The opposite is a **horizontal** slice: all the database work, then all the code, then all the tests. It looks tidy and proves nothing until the last step. A slice that only adds tests or only adds code is a defect in the brief, and `/refine` merges it into the behaviour it proves.

Each slice in the brief states four things:


| Field                | What it answers                                                                                                                     |
| :-------------------- | :----------------------------------------------------------------------------------------------------------------------------------- |
| **What it delivers** | The end-to-end behaviour, in a sentence. Not a file list: paths and line numbers rot between refine and deliver.                    |
| **ACs**              | Which acceptance criteria it makes true.                                                                                            |
| **Seam → test**      | Where it is tested and how: a unit test on the orchestration method, a live integration test on the service, a GUI spec for a flow. |
| **Blocked by**       | Which slices must be done first.                                                                                                    |


In ADO each slice is a child Task with predecessor links, so the blocking edges are visible to humans too. `/deliver` works the **frontier**: the slices whose blockers are all done, one at a time. For each one a specialist goes red then green, a verifier runs the tight loop, and the slice is committed as `AB#<id> S<n>` before its Task closes.

Slicing is where the design earns its keep, for four reasons:

- **Context.** A specialist starts each slice with a fresh context and a pointer to the brief, instead of dragging the whole story along.
- **Proof.** Every slice leaves a commit that built and passed its tests, so a failure points at one slice, not at the whole story.
- **Resumability.** `progress.md` records which slices are committed. A blocked or interrupted run resumes at the first unfinished one.
- **Review.** A human reading the PR sees small commits that each map to criteria, not one diff.

Two special cases come up. A **discovery slice** (S0) answers a question the brief depends on before anything is built, and its finding can change the brief. A wide mechanical refactor becomes expand, then migrate in batches, then contract, so every step stays shippable.

Story 132, below, was cut like this:


| Slice | What it delivered                                                                         | Blocked by |
| :----- | :----------------------------------------------------------------------------------------- | :---------- |
| S0    | A stack that boots from the root package, and the source of installed packages identified | none       |
| S1    | The database view: latest installed version per package, tested live                      | S0         |
| S2    | Config entry and Post DEE: pattern-filtered rows, safe on failure                         | S1         |
| S3    | GUI spec with screenshots                                                                 | S2         |
| S4    | Help page                                                                                 | S2         |


S3 and S4 are both blocked only by S2, so either can go next. The agent takes them in brief order.

### Stage 2: The worktree

Creating the Orca worktree named `<id>-<slug>` triggers the setup script in `orca.yaml`. The repo's devcontainer comes up for that worktree, upstream packages are restored, test run settings are written, and the worktree seeds its own MES stack from the daily backup in the background. The Agent tab then starts `claude "/deliver <id>"`.

![WorkTree](https://image.j-roque.com/posts/20261116-orca-agentic-mes/worktree.png)

### Stage 3: Deliver, the autonomous part

`/deliver` is an orchestrator. It never writes code. It keeps state in a `progress.md`, hands subagents **pointers** instead of pasted content, and never asks questions in the terminal. When it cannot proceed it stops as blocked, writes the question with a recommended answer to the work item, and ends its turn on it. Orca shows *waiting*. 

![Deliver blocked](https://image.j-roque.com/posts/20261116-orca-agentic-mes/deliver-blocked.png)

My answer resumes the run.

The slice loop is the core:

1. A specialist implements the slice, red then green, at the seam named in the brief.
2. A verifier runs build and unit tests in a tight loop, with at most three fix rounds.
3. The slice is committed with the work item id, and its Task closes.

After all slices: a merger rebases onto the base branch, a verifier runs the full DoD once (`cmf build --test`, `cmf pack` with every new file inside the package, hot deploy, live tests), and two independent reviewers go through the diff. Then ADO Test Cases are created for each automated acceptance criterion. The Publish step writes the PR text with `pr-description`, pushes, and opens a **draft** PR with an evidence table and a section on merge danger. That is the last thing before gate 2. Finally `cmf-debt` lists new shortcuts and `retro` writes proposals.

Anything that cannot run in the worktree is marked `deferred-to-CI` with a reason. It does not get quietly skipped.

### Stage 4: Review, iterate, merge, learn

Gate 2 is the PR. `/iterate-pr` reads threads and failed pipeline checks and sorts each into fix, answer or push back. Push-backs stay with me. Scope changes go back to `/refine`.

After the merge, the `retro` skill turns the run's friction into proposals: a gotcha in a skill, a hook rule, a review standard, or a deletion of a stale instruction. A delivery run never edits `.claude/` itself. Improvements ship as a normal reviewed PR, because an agent that rewrites its own rules unobserved is a problem I do not want to debug.

## A Real Walkthrough: Story 132

Everything above is design. Here is the first story I delivered entirely this way: agents did the implementation, humans decided at the gates. It is real, it is small, and it did not go smoothly. That is why it is useful.

**The story.** System Monitoring > Application > Versioning lists product components and nothing about customization. Administrators wanted the installed customization packages there, matching configurable name patterns, each with its latest installed version and install date, without database access. Three story points.


| Stage               | What happened                                                                                                                                                                      | Who decided                                                      |
| :------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------- |
| `/refine`           | Brief with 8 Given/When/Then criteria, an artifact map, slices S0 to S4 with blocking edges, and a Doubt Rule of "ask the human". Story and child Tasks published to ADO.          | Human, in grilling rounds; decisions recorded and numbered       |
| Worktree            | Branch, devcontainer, restore. Stack seeding found no daily backup.                                                                                                                | Orca script                                                      |
| `/deliver` S0       | **Blocked.** No stack could be seeded. Moving the work item to Active also failed: Story Points were required.                                                                     | Human: clean install, 3 points                                   |
| `/deliver` S0       | Plan A (a custom service querying the EnvManager) died: the 11.3.5 image serves no customization endpoint. Source became a view over the installation journal, read by a Post DEE. | Human, "for now"                                                 |
| `/deliver` S1 to S4 | View, DEE with its config entry, live integration spec, GUI spec with screenshots, Help page. Defects found only on the live stack, fixed there.                                   | Agents; human for the access token and one forced password reset |
| Review              | Two reviewers, Standards and Spec: Must 0, Should 4 each, verdict fix-then-ship. Fixes applied.                                                                                    | Agents; human for one behaviour question                         |
| Publish             | Draft PR, 6 commits, 17 files, seven ADO Test Cases linked. The agent could not push.                                                                                              | Human pushed                                                     |
| Retro               | Ten items; some became skill gotchas in a normal commit.                                                                                                                           | Human promoted them by hand                                      |




&nbsp;


{{< img src="https://image.j-roque.com/posts/20261116-orca-agentic-mes/workitem-132.png" alt="Story 132 in Azure DevOps" caption="*Story 132 as it stands after delivery: tagged `agent:in-review`, three story points, the draft PR and retro commits linked, and the slice Tasks closed.*" >}}


### Refine: the brief did its job

Every acceptance criterion is a Given/When/Then, so each one maps to a test. This is AC4, the one that is easy to forget:

```md
- **AC4** Given the request asks only for other contexts, When the service runs,
  Then no package rows are added and the view is not queried.
```

The slices carry their seam and their blocker, so the agent knows what to test and in what order:

```md
| Slice | What it delivers end to end | ACs  | Seam -> test                                     | Blocked by |
| S2    | Config entry + Post DEE: context-gated, pattern-filtered rows read from the view, safe on failure | AC1-AC6 | DEE -> unit (mocked data source); service -> Biz integration | S1 |
```

The Doubt Rule, "ask the human", is the strictest option on offer. I chose it because this was the first run, and a wrong guess costs more than a question.

### Deliver: blocked on the first slice

S0 is the slice that boots a stack. It stopped there. `/deliver` wrote the blocker, the evidence and a recommended answer, posted it to the work item, and ended its turn on the question. Trimmed:

```text
Blocked: S0. No daily backup exists for 11.3.0 to 11.3.5 under any lookup name,
so no stack can be seeded.
Decision needed:
  (a) publish a daily backup  - recommended
  (b) authorise a --nobackup boot from the root package on an empty database
  (c) accept S1 unit-only and defer the e2e to CI

tags: agent:in-progress  ->  agent:blocked
```

Orca showed *waiting*. I answered (b): clean install. The same run exposed a second snag: moving the work item to Active was rejected because Story Points were required. The agent did not invent an estimate. I set 3. Both frictions went back into the `refine`, `ado-workflow` and `deliver` skills, and the block comment is now mandatory.

### The design pivot, and a wrong explanation

The first plan was a custom service querying the EnvManager. The live stack disagreed: the 11.3.5 EnvManager image serves no customization endpoint. The agent's first explanation was wrong. It said the local EnvManager was a one-shot installer. A few entries later in `progress.md` it corrected itself: the image simply has no such controller. It had also found the real record, an installation journal table in the Online database with one row per install.

I chose a database view over that table, read by a Post DEE, "for now". The brief was rewritten, the old version kept, and the dependence on a table that is not a public contract is isolated in one view. If its shape changes, one file breaks.

This is a key part of the implementation, sometimes you think the correct path approach is one and after digging you find out you have to redesign, with this flow this is much faster to find out and to correct course.

### Live defects that no unit test could see

First I had a skill only running unit tests. Quickly I found that it was not enough so I changed my skill. The skill now says to run every DEE on the live stack before review.

The GUI spec produced the evidence for AC8. These are the two screenshots it captured on the worktree's own stack: the page with the product rows at the top, then the customization packages listed with their version and install date.

{{< img src="https://image.j-roque.com/posts/20261116-orca-agentic-mes/01-versioning-page.png" alt="Versioning page with product rows" caption="*Application / Versioning: product rows (Database, Business, Customization) with the new custom package rows starting below them.*" >}}




&nbsp;


{{< img src="https://image.j-roque.com/posts/20261116-orca-agentic-mes/02-custom-packages-listed.png" alt="Custom packages listed" caption="*Scrolled down: one row per installed customization package, each with its latest version and install date, as AC2 and AC7 require.*" >}}


### Review, publish, retro 

One finding still needed a human: should a blank pattern list fall back to a default or show nothing? The human said show nothing.

The agent then built a clean PR branch and opened the draft PR with the evidence table..

### What this proves, and what it does not

It proves that one story, in one environment, went from work item to draft PR with agents doing the implementation and humans deciding at the gates. It proves the blocked flow worked, the evidence table was real, and the retro produced changes that landed.

It does not prove more. One story. One environment. A small change of 3 story points. This first user story still had a lot of hurdles and things to learn, but by doing more and more we can aim for an ever more seamless workflow.

## The Specialists

Each package type has an agent with the skills it needs preloaded. The last column is the honest one: **agentic** means every domain skill behind the agent is confirmed; **draft** means `/refine` routes the slice to a human.


| Artifact                                                                                    | Agent                          | Why it is separate                                                                                                                         | Status                                                                         |
| :------------------------------------------------------------------------------------------- | :------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------ |
| Custom service (`Business`)                                                                 | `cmf-business-service-creator` | Service layers, API conventions, unit seams                                                                                                | agentic                                                                        |
| DEE action (`Data`)                                                                         | `cmf-dee-action-creator`       | A **DEE** (Dynamic Execution Engine) action is custom code the MES runs at a service hook; markers and extracted code have their own rules | agentic                                                                        |
| Master data, process rules, exported objects, DB scripts, reports, Grafana, Security Portal | `cmf-data-developer`           | Configuration as metadata, not code                                                                                                        | all draft, human-routed                                                        |
| GUI (`HTML`)                                                                                | `cmf-ui-builder`               | Different toolchain, different tests                                                                                                       | agentic                                                                        |
| Help                                                                                        | `cmf-ui-builder`               | Same agent, `help-docs` skill                                                                                                              | draft, human-routed                                                            |
| GUI e2e specs                                                                               | `cmf-ui-e2e-test-creator`      | Tests are authored apart from the feature                                                                                                  | agentic                                                                        |
| IoT tasks, converters, drivers, workflows                                                   | `cmf-iot-developer`            | Different runtime, mocha tests                                                                                                             | 3 of 5 skills draft (task-converter, driver, workflow); human-routed for those |
| Remaining tests, bug regressions                                                            | `cmf-test-engineer`            | Fills seam gaps                                                                                                                            | agentic                                                                        |
| Build, unit, pack, live checks                                                              | `cmf-verifier`                 | Reports pass or fail, fixes nothing                                                                                                        | agentic                                                                        |
| Standards or Spec review                                                                    | `cmf-reviewer`                 | Read-only, fresh context, one axis                                                                                                         | agentic                                                                        |
| Rebase and hot-file conflicts                                                               | `cmf-merger`                   | Resolves `cmfpackage.json`, numbered files, `.sln` by rule                                                                                 | agentic                                                                        |


Eleven of the 40 skills are draft: database-scripts, help-docs, grafana, master-data, exported-objects, security-portal, reporting, process-rules, iot-driver, iot-task-converter and iot-workflow. Those are the routine bread of an MES project: master data, reports, IoT drivers. "Agentic" here means the code-centric half. For now we've focused on the two main core elements of a project, back-end (custom services & DEEs) and custom GUIs.

Every specialist returns the same contract, in 300 words or less: `STATUS`, `CHANGED`, `TESTS`, `DOD`, `SHORTCUTS`, `RISKS`. Logs go to an evidence folder. A hand-back without the changed files and test evidence counts as failed, whatever its status line claims.

## Why the Design Looks Like This

**The orchestrator holds pointers, not content.** A long run exhausts context before it finishes. The orchestrator carries `progress.md`-level state; each specialist starts a slice with fresh context and reads only what it was pointed at. A specialist that "could not find" something almost always lacked a pointer, so the fix is a pointer, not more prose.

**The author does not grade its own work.** Verification is a different agent from implementation. Review is two more agents, one on the Standards axis and one on the Spec axis, read-only, each with an independent context. Each finding needs a concrete case, and findings the reviewer cannot back are dropped.

The limit is real. The reviewers share a model family and receive the same injected rules as the author, so their blind spots correlate with the author's. Independence of context is not independence of judgement. I treat it as an assumption. In story 132 the two reviewers raised four Should findings each and the fixes were real, but I cannot show the author would have missed them. The human PR review stays for that reason.

**Hooks catch the common mistakes deterministically.** Rules written in a skill get ignored under pressure. `hooks/guard.sh` runs before Bash, Edit, Write, MultiEdit, NotebookEdit and Read calls and blocks:

- toolchain commands (`dotnet`, `cmf`, `npm`, `node` and friends) when the program is absent from the current shell, with a pointer to the devcontainer wrapper
- force pushes, `reset --hard`, and pushes to the base branch or environment branches
- printing secret-bearing files such as run settings, environment configs or auth files, by a list of printer command names
- Edit and Write under `Dependencies/`, the restored upstream packages

It is a seatbelt, not a security boundary. The known gaps are in the unproven section below. A second hook, `inject-rules.sh`, fires when a subagent starts and injects the shared core rules, because subagents do not inherit your session context. Without it, the carefully tuned orchestrator is surrounded by specialists who have never heard of the rules.

**Unverified skills route to humans.** A skill with `metadata.status: draft` sends its slices to a human instead of an agent. That is the table above.

**The system has its own cost.** 40 skills, 10 agents, hooks and evals are a codebase. They need maintenance, and a stale skill misleads agents with full confidence. The retro loop and the lint script exist because I expect that cost to be real.

## How It Is Tested

Skills are code, so they get tests. A lint script enforces the authoring standard: names match folders, descriptions have a length limit, no project literals, links resolve. `guard.sh` has its own test script and a file of block and allow cases.

Trigger evals check that a skill activates on the prompts it should. They cover 36 skill files for 40 skills. **None exist for `refine`, `deliver`, `iterate-pr` or `cmf-setup`**, which are the four user-invoked workflows.

The golden set is five pre-refined briefs, one per artifact family, delivered headless against a fixture repo and checked mechanically: commits touched the right package types, tests were added, build and pack evidence passed, no Must-fix was left open. It runs in offline mode: no ADO, no push, no PR. So the tags, ADO Test Cases, draft PR, blocked and waiting flow and `/iterate-pr` are **untested**. The fixture itself is not in the folder: the README tells you to write a `fixture.env` pointing at a repo and commit, and that file is not there. Two of the five cases, `g4-iot` and `g5-master-data`, cover draft families, so a green run there is weaker evidence.

I am not claiming that golden runs have been executed at any volume. What exists is a smoke test of the build and verify stages, ready to run.

## Sources

None of this started from a blank page. Two public sources shaped the discipline skills, and I adapted them to the MES and CM context.

From Matt Pocock's [skills collection](https://github.com/mattpocock/skills) I retained the skill names `grilling` and `diagnosing-bugs`, and adapted the ideas for CMF and MES: tracer-bullet vertical slicing and red/green TDD at agreed seams (`tdd-cmf` maps each seam to a test type). `grilling` interviews the human: it maps open decisions as a design tree, answers whatever the repo can answer, and asks the whole frontier in one round with a recommended answer per question. `diagnosing-bugs` makes a deterministic red feedback loop the first step.

From Dietrich Gebert's [Ponytail](https://github.com/DietrichGebert/ponytail) I retained the "laziest senior developer" framing in `lean-change`: climb a ladder (not needed, already in the repo, a platform feature, an installed dependency, one line, minimum code) and take the first rung that fully works. In CM terms, a configuration entry, a smart table (a configurable lookup the MES reads at runtime) or an existing service often beats a new class. The `// shortcut: <limit>, <when to upgrade>` marker, harvested by `cmf-debt`, belongs to the same lineage.

## What Is Still Unproven

This is the honest section. It is iteration one.

**Draft skills.** The 11 skills above were written from existing knowledge base. I have not confirmed them against enough real repositories. Until a human does and removes the draft flag, `/refine` routes those slices to a human. This is the system working as designed, and it also means large areas of a real project are not agentic yet. Everything behind `cmf-data-developer` is in that state.

**Not battle-tested.** See the testing section: one real story delivered end to end (see the walkthrough), no golden run volume claimed, no cycle-time data, no review-reject rate, no proof that a reviewer caught what the author would have missed.

**Gotchas I have already paid for.** They live in the `deliver` skill's Gotchas section now, which is the point of the retro loop. The first one below comes from the story 132 retro:

- The session's SSH agent socket is fixed when it starts. An agent restored later stays invisible to it, so the push step fails late. The skill checks early and hands the human the exact commands.
- Subagent hand-backs can render `<`, `>` and `&` as HTML entities, which looks like file corruption. Check the file on disk before starting a fix loop.
- `git fetch` and `git push` can outlast the tool timeout and get moved to the background mid-command. Start them in the background and wait for the notification.
- `git add <stale path> ; git commit` commits whatever was already staged. Chain with `&&` and check the commit's file list afterward.

### Agents in a regulated manufacturing context

Manufacturing software often lands in regulated environments. An assessor would reach for **GAMP 5** (2nd edition) for risk-based computerised system validation, the FDA's computer software assurance guidance for how much testing effort a tool or change deserves, and **21 CFR Part 11** for electronic records and signatures. They would ask:

- Who is the author of an agent-written change, and who is accountable for it?
- What is the audit trail from requirement to merged code, and does it survive the agent's session?
- Was the tooling itself qualified, or at least assessed for its intended use?

The design leaves a trail: a brief approved by a person, commits tied to work item ids, evidence logs, Test Cases linked to acceptance criteria, two review reports and a PR a human merges. **This workflow does not claim compliance.** Whether that trail answers the questions above is open, and a matter for your quality people, early.

### Where humans stay

Humans own the brief and every decision in it, the PR review and the merge, the push-backs on review comments, any blocked question, every draft-skill slice, and the promotion of retro proposals into the environment. They also handle anything touching upstream packages, which agents cannot edit and report as cross-repo work.

## What Comes Next

- Confirming the draft skills against real repositories, one at a time, until the flag disappears. This is the item that moves the thesis most.
- Trigger evals for the four workflows, a golden mode that exercises the ADO and PR path, and a second fixture repo.
- Making the retro loop repeatable, so proposals reach `.claude/` with less ceremony but the same review.
- Real parallelism: several worktrees delivering at once, with the human gates as the only queue.
- Measuring cycle time and review-reject rate, so the next post can have numbers instead of opinions.

## Final Thoughts

The model is not the hard part. The hard part is the environment around it: isolated worktrees, a catalog that maps requirements to specialists, hooks that catch the common mistakes, reviewers with their own context, and gates placed where a human judgement is actually needed.

Iteration one delivered one real story, a DEE, a view, tests and a Help page, with agents between the gates. That is a first data point, not a result. It does not yet cover the data and IoT families, and it is not proven at volume. I will report back when there are numbers.

> An agent can build the project. Deciding what is worth building, and whether it is right, is still the job.