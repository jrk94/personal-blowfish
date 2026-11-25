---
title: "Human in the Loop for MES Agents: Notifications, a Respond Wizard and Reruns with the Answers"
description: "An MES v12 agent that returns needs_review now opens a notification, holds a conversation with the person who answers, and runs again with their answers, all from one Connect IoT task."
summary: "Last time the agent mapped equipment files to a Data Collection and sent the doubtful ones to a human. This post builds that human part: a notification with the agent's open questions, a chat-style Respond wizard, request/reply delivery over the message bus, and a single Run Agent task that replaces the code tasks and the second workflow of the first version."
categories: ["AI"]
tags: ["MES v12", "AI Agents", "Human in the Loop", "Connect IoT", "Notifications"]
date: 2026-11-25
showInRSS: true
draft: false
authors:
  - Roque
---
If the agent cannot say it is unsure, it guesses, and a guessed value that is posted to a Data Collection stays there.

Last week's agent returns `needs_review` when it is unsure. This post shows what happens next, from the notification to the person's answer and the agent's second run.

---

## Overview

In [the previous post](https://j-roque.com/posts/20261118-agenticframework/) a `Connect IoT` workflow read equipment files, a "File Parser for DataCollection" agent mapped them to a Data Collection, and a guard clause routed the answer to `ok`, `needs_review` or `rejected`. `needs_review` went "to a human". I did not show that part, because it did not exist yet.

It exists now. It was built with the agentic process from [Orca, Worktrees and the First Iteration of an Agentic MES Project](https://j-roque.com/posts/20261116-orca-agentic-mes/), and the walkthrough below is one real run on a local MES v12 stack: one ambiguous file, the real agent, two rounds of answers.

## An Ambiguous File

The test case is a rework re-inspection of housing `A12`, serial `SN-0042822`. The file is ambiguous in two ways:

- Hole A appears twice: `12.024` as found with burr, and `12.013` re-measured after deburring.
- Roundness `4.6` and flatness `22` were read with dial indicator `DI-2`, and nobody recorded its unit.

A parser would pick one. The agent returns `needs_review` with open questions instead.

## The Request Is an MES Notification

The request is a standard MES Notification of a custom type, `HumanInTheLoopNotify`, named `HITL-<GUID>`. The GUID is the correlation id: the `Run Agent` task keeps the request under that id in its persistent store until someone answers.

The Title is readable ("Review needed: DC_Housing_A12_FullInspection – no material"), and the Details are HTML: why a review is needed, the open questions, what is missing, the agent's notes, the Data Collection and what to do.

The full agent response and the original prompt live in a payload attribute. An attribute value holds at most 512 characters, so the payload is chunked and shown as two tiles, *Agent Feedback* and *Original Prompt*, among the stock notification Details tiles.

| ![The HITL notification](https://image.j-roque.com/posts/20261125-humanintheloop/notification-details.png) |
|:--:|
| *The notification page: Respond in the ribbon, the agent's two open questions, and the Original Prompt and Agent Feedback tiles.* |

This needs no new entity or page, because Notifications already have assignment, a side panel, history and clearing.

## The Respond Wizard Is a Conversation

**Respond** is a custom action on the Notification page and in the notifications side panel. It opens a wizard that looks like a chat.

Each agent request is a formatted bubble. Each answer is a blue bubble with who sent it and when. All of it is rebuilt from the notification's standard MES history, so there is no extra storage.

| ![Respond wizard, round 1](https://image.j-roque.com/posts/20261125-humanintheloop/respond-round1.png) |
|:--:|
| *Round 1: the agent's request as the first bubble, an answer box, and three outcomes.* |

The three outcomes:

| Button | What it does |
| --- | --- |
| **Answer** | Sends the text, keeps the notification open, the agent runs again |
| **Submit** | Accepts the agent's current result as it is |
| **Discard** | Ends the request, after a confirmation |

Here is the answer for round 1 before it is sent:

| ![Answer typed in the wizard](https://image.j-roque.com/posts/20261125-humanintheloop/answer-typed.png) |
|:--:|
| *A person answers the way they would answer a colleague.* |

## Delivery Uses Request/Reply on the Message Bus

The simplest design saves the answer and then publishes a message. With it, the agent can run on an answer the MES never saved, or the MES can save an answer that nobody received.

So a DEE on the notification save sends the outcome over the MES message bus as a **request** to the subject `<base topic>.<GUID>`, and saves the answer only when the requester replies `{"status":"ack"}` within 30 seconds. Without the acknowledgement, the message shows in red with the requester's error and a **Resend** button.

After an acknowledged answer the notification stays open. The title gets the prefix "Waiting for agent reply:", the buttons are disabled, the answer box is hidden and the wizard refreshes itself until the agent answers.

| ![Waiting for the agent](https://image.j-roque.com/posts/20261125-humanintheloop/answer-waiting.png) |
|:--:|
| *Answer acknowledged: the blue bubble with the author and time, "Waiting for the agent's reply…", buttons disabled.* |

## The Agent Runs Again with the Whole Conversation

The agent is stateless, so the task gives it everything: the original prompt plus a `review` object with the round, the previous result and every answer so far.

```typescript
export function nextPrompt(pending: PendingRequest, answer: string, by: string): any {
    const prompt: any = pending.prompt !== null && typeof pending.prompt === "object" ? { ...pending.prompt } : { prompt: pending.prompt };
    const stored: any = prompt.review;
    const answers: ReviewAnswer[] = Array.isArray(stored) ? stored.slice() : Array.isArray(stored?.answers) ? stored.answers.slice() : [];
    const round: number = (typeof stored?.round === "number" ? stored.round : answers.length) + 1;
    answers.push({ round, answer, by });
    prompt.review = { round, previousResult: pending.reply, answers };
    return prompt;
}
```

If the new result still needs review, the **same** notification is updated: the title becomes "Review needed (round 2): …", the outcome resets, and the conversation shows round 1 request, answer, round 2 request.

| ![Round 2 of the conversation](https://image.j-roque.com/posts/20261125-humanintheloop/conversation-round2.png) |
|:--:|
| *The end of the round 1 request, the blue answer, and the round 2 request.* |

In the live run, the round 1 answer was *"Hole A is Bore D1. Post BoreDiameter1 = 12.013, the value re-measured after deburring; ignore 12.024"*. The agent took it and asked about the unit of `DI-2`. The round 2 answer was *"DI-2 was set to µm: BoreD1Roundness = 0.0046 mm and Flatness = 0.022 mm"*, plus the remaining confirmations.

Then the result was accepted and the notification cleared itself. The Data Collection instance `DC_Housing_A12_FullInspection-000000007` was posted with `BoreDiameter1 = 12.013` and `Bore D1 Roundness = 0.0046`. Both values come from the person's answers.

| ![The posted Data Collection instance](https://image.j-roque.com/posts/20261125-humanintheloop/dc-instance-data.png) |
|:--:|
| *The posted instance: Bore D1 Diameter 12.013 and Bore D1 Roundness 0.0046.* |

## One Run Agent Task Replaces the Request and Reply Workflow

The first version of this flow used ten workflow pieces: a code task to build the request, the Create Notification task, a second workflow with a Message Bus Listener, a Switch, two code tasks, a Reply and a Call SubWorkflow, and two more code tasks to forget or end a request. It worked, but every piece had to be maintained separately, and the state was spread across them.

Now it is one custom task, `Run Agent`, from the [task library](https://j-roque.com/posts/20260702-codetasktocustomtask/) that already called the agent. It has two optional features, each in its own settings tab with an enable toggle. The Agent Flow is now `SubWorkflow Start`, `Run Agent`, then `Accepted` to the Data Collection and `Rejected` to Log and Move file.

| ![Agent Flow with Run Agent](https://image.j-roque.com/posts/20261125-humanintheloop/agent-flow-run-agent.png) |
|:--:|
| *Outputs: Response, Prompt, Accepted, Rejected, Notification and Ended.* |

### Guard Clause tab

The code-task guard from last week is now three JSON Schemas, validated with `ajv`:

- **Response Schema**: the shape every response must have. If it does not match, the result is Rejected with "Invalid agent response: …".
- **Reject Schema**: for example `status = error`. Rejected, with the agent's reason.
- **Accept Schema**: `status` is `ok`, nothing is missing, every mapping has high confidence, there is some data. Accepted.

Anything valid that is not accepted needs review.

```typescript
public judge(raw: unknown): Verdict {
    const result: any = parseJson(raw);
    if (result === null || typeof result !== "object") {
        return { kind: "rejected", result: raw, reason: "Invalid agent response: it is not a JSON object" };
    }
    if (this.response !== undefined && !this.response(result)) {
        return { kind: "rejected", result, reason: `Invalid agent response: ${errorsText(this.response)}` };
    }
    if (this.reject !== undefined && this.reject(result)) {
        const said: unknown = (result as { reason?: unknown }).reason;
        const reason = typeof said === "string" && said.trim() !== "" ? said : "The agent could not do the job";
        return { kind: "rejected", result, reason };
    }
    if (this.accept!(result)) {
        return { kind: "accepted", result };
    }
    return { kind: "review", result };
}
```

The Accept Schema holds the old `effectiveStatus` rules as data:

```json
{
  "type": "object",
  "required": ["status", "rawData", "mappings", "missing"],
  "properties": {
    "status": { "const": "ok" },
    "missing": { "type": "array", "maxItems": 0 },
    "mappings": { "type": "array", "items": { "properties": { "confidence": { "const": "high" } } } },
    "rawData": { "type": "object", "minProperties": 1 }
  }
}
```

One rule did not survive the move: the old function also checked that every mapped parameter has a value in `rawData`. JSON Schema cannot compare a value in one part of the document with the keys of another, so that check was dropped. The Accept Schema still requires the agent's `missing` list to be empty, and the MES still validates the Data Collection post.

The rules of the guard are now a task setting that a process engineer can read and change without a code deploy.

| ![Guard Clause settings](https://image.j-roque.com/posts/20261125-humanintheloop/settings-guard-clause.png) |
|:--:|
| *Guard Clause tab: the toggle and the three schema editors.* |

### Human In The Loop tab

Settings: Requester, Notification Type, Assignment To (Everyone or a Role), Base Topic and Store Key. With it enabled, the task creates and updates the notification itself, subscribes once to `<base topic>.>`, acknowledges, runs the agent again, clears on accept or reject, and treats Submit as "accept the reviewed result without running again".

Pending requests live in the controller's persistent data store, so they survive a restart. Without the guard clause, every response goes to a person.

| ![Human In The Loop settings](https://image.j-roque.com/posts/20261125-humanintheloop/settings-human-in-the-loop.png) |
|:--:|
| *Human In The Loop tab: requester, notification type, assignment, base topic and store key.* |

Inside the message handler, the task stores the new prompt, acknowledges, and only then runs the agent:

```typescript
case "answered": {
    const prompt = stripNulls(nextPrompt(request, message.response ?? "", message.respondedBy ?? ""));
    pending.set(correlationId, { ...request, prompt });
    await this.storePending(pending);
    answer({ status: "ack" });
    await this.inNewContext(() => this.rerun(correlationId, request.agentName ?? this.agentNameOf(this.agentName), prompt));
    return;
}
```

The agent takes longer than the 30 seconds the MES waits, so the acknowledgement cannot wait for it. Because the pending state is stored before the acknowledgement, an acknowledged answer is not lost if the controller stops.

A `Prompt` output emits the prompt the result belongs to. The Data Collection task takes its resource from it, so the data is built from the original request even after several human rounds.

## What the Human Step Does Not Guarantee

The acknowledgement is a delivery guarantee. The MES saves the answer, and the agent runs on the same answer. Nothing checks that the answer is right.

A few limits are worth stating plainly:

- **A confidently wrong agent never reaches a person.** Escalation depends on the agent's own `status` and confidences. An `ok` with high confidence and wrong values passes the Accept Schema. The MES limits and SPC on the Data Collection are the next line.
- **Submit is one click.** It posts the result the person reviewed, including a `needs_review` result. That is intended (the person vouches for it), and the notification history records who did it and when, but it is also how review fatigue turns into rubber-stamping.
- **There is no round cap.** The person decides when to stop answering and submit or discard. The first version had a cap; the merged task does not yet.
- **An unanswered request waits.** There is no expiry or escalation. The notification stays open, and the file stays in the folder.
- **This is one run.** One file, two rounds, one person. I have no numbers yet on how often the agent escalates or how long people take to answer.

For approval flows with more structure than free text, [Business Scenarios](https://j-roque.com/posts/20260611-businessscenario/) remain the better fit.

## What Went Wrong on the Way

Lessons from the live runs:

1. **The message bus reply must be `{ content: … }`.** A bare object arrives empty on the MES side. When that happened, the MES correctly refused to save the answer, but the task had already run the agent again on it, which is the exact failure the request/reply design is meant to prevent. The reply shape is now covered by a unit test.
2. **Sub-workflow tasks are instantiated per call.** Each file starts its own copy of the Agent Flow, and each copy of `Run Agent` subscribed to the bus. With two files processed, one answer ran the agent twice. Now one copy per controller listens.
3. **`stringToEntityInstance` turns a missing material into an empty entity.** For optional entities, use the stock `jsonata` converter.
4. **Attribute values are capped at 512 characters.** The payload is chunked, and the answer is limited to 512 characters.
5. **Ship the task library the platform way.** Pack it with `packagePacker`, upload it to the automation repository, terminate the old library entity if the version is the same, then synchronize.

## How It Was Built: ADO Story 196

The feature is Azure DevOps User Story 196. It was refined with an agent in a conversation, sliced into S0 to S5, and delivered by agents in a worktree until it reached `agent:in-review`, the process described in the [Orca post](https://j-roque.com/posts/20261116-orca-agentic-mes/).

| ![ADO User Story 196](https://image.j-roque.com/posts/20261125-humanintheloop/ado-story-196.png) |
|:--:|
| *The story with its acceptance criteria and the child Tasks S0 to S5, all closed by agents.* |

The first version answered once. After using it, I wanted a conversation, so the story got a second iteration, recorded in its description next to the original brief.

| ![ADO story, Iteration 2](https://image.j-roque.com/posts/20261125-humanintheloop/ado-story-196-iteration2.png) |
|:--:|
| *"Iteration 2 (2026-11-25): a conversation instead of a single answer", recorded in the story after the change.* |

## Final Thoughts

The guard clause decides which results the agent may post on its own. For the others, the agent asks a person, and the MES saves the answer only after the workflow acknowledges it.

In the next post I'll give the agent a memory across requests: it will reuse earlier answers and decisions, so nobody is asked the same question twice.
