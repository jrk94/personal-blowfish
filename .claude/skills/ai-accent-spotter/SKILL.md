---
name: ai-accent-spotter
description: Spots "AI accent" in a blog post (slogan lines, not-X-but-Y, staccato fragments, sentences that announce their own importance, decorative metaphors, slogan headings) and suggests plain rewrites in Roque's own voice. Use when the user asks to check a post for AI tone, make it sound less AI or more human, or "de-AI" a draft under content/posts/. It only spots and suggests. It does not check facts, structure or argument (that is blog-reviewer's job).
argument-hint: <path to index.md, or a post slug>
allowed-tools: Read, Glob, Grep, Edit
---

# AI Accent Spotter

You have one job: find the sentences in a post that read as machine-written and suggest a plain replacement for each one, in the voice of the rest of the blog.

Do not review the argument, structure, evidence or technical accuracy. Do not suggest new content. Do not rewrite sentences that are already fine. A short, accurate list is better than a long one.

## Inputs

- `$ARGUMENTS` is a path to a post (`content/posts/<slug>/index.md`) or a slug. If it is empty, ask which post, or use the post changed most recently in `git status`.
- Read the whole post before flagging anything.
- Read [examples.md](examples.md). It is a real review from the CEO of a post on this blog, and it is the calibration for what counts as AI accent and how plain the rewrite should be.
- If you are unsure what the author's own voice sounds like, read one or two of the older posts listed under "Voice reference" below.

## What not to touch

- Front matter values other than `title`, `description` and `summary`. Those three are held to the same standard as the body, and they often carry the post's slogan.
- Code blocks, inline code, equipment/file samples, log output, URLs, image captions that only name a screenshot.
- Quotes and attributed paraphrases of other people ("Francisco's point was…").
- Spelling inside sample data (for example "Centre D1" in a CMM report stays, because it is equipment output).
- Technical terms, domain jargon and product names (`Connect IoT`, `Data Collection`, `MES`, "envelope" for a process window).

## The traps

Flag a sentence when it matches one of these. Use the trap name in the "Trap" column, and combine names when more than one applies (for example "Not X but Y, metaphor").

### Sentence shapes

| Trap | What it looks like | Typical fix |
| --- | --- | --- |
| **Not X but Y / Not X, it's Y** | "Not with a smarter parser, but with...", "The output is a contract, not prose.", "The promise was never outsourcing. It was…", "It is a seatbelt, not a security boundary." | State only the Y, plainly. See "When a contrast is real" below for when the X stays. |
| **Reversal / chiasmus** | "The parser is never the hard part. Keeping twenty of them alive is." "X stops being a nice-to-have and becomes the design." "Facts that go stale in prose do not go stale when you query them." "Independence of context is not independence of judgement." | Say the actual claim, or the mechanism behind it, as one declarative sentence. |
| **Slogan / aphorism** | A short, quotable line that sums up the paragraph: "Flexible on the input side, strict on the output side." "The MES decides what is true." "Build what makes you different. Buy what makes you the same." | Replace with a literal sentence that says what happens, or delete it if the paragraph already said it. |
| **Slogan pair / parallel staccato** | Two mirrored sentences built to be quoted: "The model reads the chaos. The MES decides what is true." "Organizations are reasonably good judges of their problems. They are poor judges of their solutions." | Merge into one sentence with a connector ("and", "because", "so"), or delete. |
| **Staccato run** | Three or more very short sentences or fragments in a row: "One place to define agents. One place to grant tools. One runtime." "Lock-in is real. Shelfware is real." "One story. One environment." | One sentence with a list. |
| **Fragment** | A verbless sentence used for punch: "No prompt edit, no redeploy." "Not absent, placed." "Same shape." | Make it a full sentence with a subject and a verb, or delete it. |
| **"That is X" equation / echo** | A sentence that names what the previous one was: "That is the shape of telemetry." "That is a catalog, and a catalog is something you can hand to agents." "That is the question the audit answers." | Fold it into the previous sentence with "which" or "so", or delete it. |
| **Tautology** | "The lead time of the supplier is still the lead time of the supplier." | State the consequence: "the order waits for the supplier." |
| **Colon reveal** | A short set-up, a colon, then the point: "The failure is slower: the requirement moves." "The pattern has been consistent: build, struggle, buy." | Drop the set-up and state the fact. |
| **Ironic or punchline closer** | Last sentence of a paragraph written to land: "Together they are ungovernable." "Only the guard gets stricter." "…and it shows." "None of these kill the idea. They define where it starts." | Fold the point into the previous sentence, or delete it. |
| **"Do X and Y" kicker** | "Change the Data Collection and the agent follows." "Ask an agent to build what you describe, and it will build exactly what you describe." | "When someone changes X, Y happens on the next run." |
| **Emphasis adverb in commas** | "re-checks the answer, deterministically, before…" | Fold it into a noun phrase: "with fixed rules". |
| **Trailing -ing commentary** | A participle clause tacked onto a fact to tell the reader what it means: "…, highlighting the importance of traceability." "…, ensuring the data stays consistent." "…, making it easier to scale." | Delete the clause, or turn it into its own sentence with the concrete consequence and who gets it. Keep -ing clauses that describe an action ("…, logging each reading to the Data Collection"). |
| **Rule of three by reflex** | Triplets that exist for rhythm, or mirror each other: "configured, secured and shipped", "maps what is there… reports what is not there", "software that decides, robots that execute, and an MES that checks". | Flag only when an item is padding or the three mirror each other. Name the real fields or actions. If all three are true and needed, leave it. |

### Framing sentences

| Trap | What it looks like | Typical fix |
| --- | --- | --- |
| **Announces importance** | The sentence tells the reader something matters instead of saying it: "This is the part that is easy to undersell." "The important design choice is...", "That is the whole trick.", "The progression is deliberate.", "…and the difference is the whole point of this post.", "This is the item that moves the thesis most." | Delete the announcement and keep the content. |
| **Setup sentence** | "Note what the log does not contain: the prompt." "This is that argument with a working integration behind it." "I think this is tractable for one specific reason." "I have one specific question for you." | Say the fact directly. |
| **Staged objection** | "There is a fair objection here." "The counterargument is real:" "The limit is real." "It is a narrower promise than that." | State the objection or the limit as a plain fact, without the stage directions. |
| **Honesty label** | "an honest list", "The last column is the honest one", "This is the honest section." "the only honest answer is…" | Delete the label. The content shows whether it is honest. |
| **Counted setup** | "Two things the table hides." "Four objections are worth taking seriously." Often the count is wrong. | Drop the count, or check it matches what follows. |
| **Abstract topic sentence** | "What does not change across the three is the centre." "What v12 changes is where the intelligence lives." "Today the economics decide how far this goes." | Start with the concrete subject: "In all three cases, agents are defined in one place..." |
| **Vague attribution** | Opinion credited to an unnamed crowd: "Experts agree…", "Many teams have found…", "Industry observers note…", "It is widely accepted that…". | Name the source, say "I think" / "in our projects", or delete. Do not invent a source; if one is needed, write "(source? confirm with author)". |
| **Heading echo** | The first sentence of a section repeats its heading: "## Deploying the Agent" followed by "Deploying the agent is the next step." | Start with the first real fact of the section. |
| **Abstract-noun fog** | A sentence of abstract nouns where nothing does anything: "Value is only accelerated on a foundation that enforces constraints." | Name who does what to what. If you cannot without inventing facts, flag it and ask the author. |

### Word choice

| Trap | What it looks like | Typical fix |
| --- | --- | --- |
| **Decorative metaphor** | Verbs and nouns borrowed from another world: *taming, attack, speaks its own dialect, closes the loop, where the pattern lives, the final word, crack, drifting into, reads the chaos, in miniature, the price of, wiring it up costs, ripples into, earns its keep, a time bomb, collides with, conjures, trample, rot*. | Use the literal verb: *reads, fails, is stored, validates, runs, needs*. |
| **Personified system** | A system that wants, cares, judges or referees: "The MES does not care about any of that. It wants…", "The MES is the referee.", "guardrails bite", "its own idea of who is allowed". | Say what the system requires, rejects or checks: "The MES needs…", "The MES rejects…". |
| **Buzz idiom / inflated stakes** | "the missing link", "in the Age of AI", "paradigm shift", "has never mattered more", "no-brainer", "behind the scenes", "the whole point", "Stay tuned for more!", "Let's dive in", "game-changer", "it's worth noting", "None of this started from a blank page." | Delete, or state the concrete consequence or next step. |
| **Inflated significance / sales language** | Ordinary facts dressed as milestones or brochure copy: "a pivotal moment for manufacturing", "marks a significant shift", "a testament to", "plays a crucial role", "boasts", "nestled in", "cutting-edge", "best-in-class", "powerful" or "rich" as bare praise. | Say what the thing does or changed, with a number or an example if the post has one. |
| **AI vocabulary** | Words that are rare in the author's older posts and common in model output: *delve, underscore, showcase* (as a verb), *leverage* (as a verb), *harness, foster, streamline, seamless, robust* (as praise), *unlock, empower, elevate, landscape, realm, tapestry, intricate, multifaceted, holistic, crucial, vital, key* (as an adjective for "important"), *additionally, furthermore, moreover* (opening a sentence). | The plain word: *look at, show, use, help, smooth, reliable, area, important*, or delete. One occurrence in a long post is a light flag; a cluster is a real one. "Robust" with a technical meaning (robust to noise) stays. |
| **Copula avoidance** | "serves as", "stands as", "acts as", "functions as", "features", "offers", "represents" used where the sentence means *is* or *has*: "The Data Collection serves as the contract." "v12 features an agent framework." | "is" / "has": "The Data Collection is the contract." "v12 has an agent framework." Keep the verb when it is literal ("the gateway acts as a proxy" in a networking sense). |
| **Slogan heading / slogan title** | Headings with a catchy half or a metaphor: "The Agent: Ask the MES, Don't Hard-Code It", "Deterministic Around the Probabilistic", "Boundaries Are the Point", "Where the guardrails bite", "Infinite features, no one in charge". Titles with "Taming", "Unlocking", "in the Age of AI". | A heading that says what the section shows: "The agent reads the Data Collection definition from the MES at runtime". Titles name the concrete thing being done. |
| **Em dash habit** | Em dashes used as a rhythm device, especially more than one or two per post. | Comma, period, or parentheses. |
| **UK spelling** | millimetres, micrometres, centre, judgement, colour, behaviour, labour, catalogue, modelled, optimise, computerised | US spelling. Not inside sample data, code or log output. |

### Whole-post patterns

Check these after the line-by-line pass. They are invisible one sentence at a time.

| Trap | What it looks like | Typical fix |
| --- | --- | --- |
| **Repeated thesis** | The post's main contrast restated as a slogan in several places: description, summary, opener, a blockquote and the closer all say "AI made writing cheap, not owning". | Keep it once, in the plainest place (usually the opener or Final Thoughts), and flag the repeats. |
| **Recurring motif** | The same figure across sections: "wall" three times, "close the loop" twice, "speak to each other as fluently" twice. | Use the literal term each time, or keep the figure once where it is introduced. |
| **Repeated openings** | Several paragraphs or consecutive sentences that start the same way ("This means…", "The result…", "In short…", "What this gives us…"), or a section where every sentence has the same length and shape. | Name the run in one row and suggest varied openings for the worst two or three; do not rewrite every sentence. Steps in a numbered walkthrough that all start with a verb are fine. |

## When a contrast is real

"Not X but Y" is the most common trap, and also a normal way to correct a reader. Keep the X when:

- It is a **disclaimer**: "This is an illustration, not a description of a deployed system."
- It corrects a **misreading a reader would plausibly make**: "Tools are granted per agent, not per prompt." "I read the state of a run from the card, not from a terminal."
- It is the post's **thesis, stated once, plainly**. The repeats are the trap (see "Repeated thesis").

Flag it when the X is a straw man nobody holds ("The agent is one task in a workflow, not the workflow"), when it is decoration ("a contract, not prose"), or when the same contrast already appeared earlier in the post.

## Patterns that are fine

Do not flag these:

- A rhetorical question followed by a direct, literal answer ("Is this always a problem? No, most integrations don't..."). The author uses this in posts written before AI tools. If the answer is a slogan, flag the answer.
- A personal anecdote, a joke about something that actually happened, or "I"/"we" sentences about what the team actually did. Figures of speech are fine in an opening or an anecdote ("demon speak", "Before AI sucked the oxygen out of every conversation"). In an explanation of how something works, they are a trap.
- A metaphor the post introduces, defines and builds on (for example a "skeleton" or "puzzle" that the post then explains). Flag only figures that appear once for effect.
- "Let's" before a hands-on step ("Let's create our workspace").
- Bold on the first use of a key term.
- Passive voice and a system as the subject when that is the accurate description ("The agent definition is stored, secured and versioned in the MES", "The MES rejects the reading"). Generic anti-slop guides flag these; this blog's technical writing needs them.
- Title Case headings. That is the blog's current house style.
- A blockquote that holds a fact, a quote or a stated opinion. Flag only blockquotes that hold a slogan.
- A short descriptive lead that introduces a sample or a screenshot ("A fixed-width report built for a person to read.").
- Short sentences that list real branches or rules ("`ok` goes to the Data Collection task. `needs_review` goes to a human. Anything else is rejected with a reason.").
- Two plain sentences that each state a fact, even if they happen to be parallel. The CEO's own rewrites in [examples.md](examples.md) do this ("An agent configured by hand exists in one environment only. In source control, it is versioned and installed like the rest of the project."). A slogan pair is built to be quoted; a factual pair is not.
- A plain descriptive heading, and the blog's standard section names ("Overview", "Final Thoughts"). A bare topic heading such as "The Result" can be flagged lightly, with a suggestion that states the result.
- A sentence that is a little informal or uses a common word with a light figurative sense ("handle", "reach", "on purpose"). Flag figurative language only when it replaces the literal fact.

## Slips are a human signal

Paragraphs with grammar slips, odd prepositions or non-native phrasing ("show case", "loopsback", "the one's", "rabbit holes we can go to") are usually the author's own writing, and they are the least AI part of the post. Do not rewrite them for tone. Fix only the slip itself, and collect all slips into one low-priority row at the end of the table instead of interleaving them.

## Voice reference

The target is the voice of the posts written before 2026, for example:

- `content/posts/20250424-somethingbetterthannothing/index.md`
- `content/posts/20250617-sqlite-i/index.md`
- `content/posts/20250227-secsgem/index.md`

In those posts the author:

- Writes plain declarative sentences that say what a thing is and what it does: "Connect IoT is a nodejs application."
- Explains with a concrete case: "One example that always comes to mind is machines that don't know the name of the material..."
- Uses "I", "we" and "our" naturally, and walks the reader through steps.
- Puts humor in situations and stories, never in slogan lines.
- Gives headings that say what the section does: "Create an Automation Protocol", "Setting up our Development Workspace".
- Ends sections when the content ends, without a closing punchline.

The rewrite should sound like that author on a good day: literal, specific, a little longer if it needs to be, with no line that was written to be quoted. Keep the author's facts and opinions. Do not make the rewrite more formal, and do not add hedging ("might", "could potentially").

## Process

1. Read the post and [examples.md](examples.md).
2. Go through the post top to bottom, including `title`, `description` and `summary`. For each sentence or heading that matches a trap, write one row. Quote the exact text (shorten long quotes with "…" but keep the trap words). Bold the words that cause the trap in the "As written" column.
3. For each row, write the replacement sentence in full, or "Delete." with a short reason (for example "Row 7 already makes the point.").
   - One row per point, not per sentence. If two or three adjacent sentences make the same move (a fragment followed by its reversal, a slogan pair), merge them into one row with one rewrite.
   - Do not flag a heading that already says what the section shows ("The guard clause", "What v12 Brings: Agents as MES Entities"). Flag a heading only when part of it is a slogan or a metaphor.
4. Do the whole-post pass: look for a repeated thesis and recurring motifs, and add rows for them.
5. Before you finish, re-read every suggested change and check that it does not contain a trap itself. Rewrites that swap one slogan for another are the most common failure. In particular:
   - A rewrite must not keep the contrast or reversal shape with new words. "Each one makes sense on its own, but nobody can manage all of them together", "Writing a parser is easy, but maintaining twenty is hard" and "a finding instead of a guess" are still the trap. State the concrete fact instead: what it costs, what is missing, what happens ("each with its own credentials and no single place to govern them", "most of the cost comes from maintaining the parsers").
   - When the original is a contrast you cannot rephrase without keeping the shape, prefer "Delete." if the paragraph already makes the point.
   - A rewrite must not end in a new triplet or a new closer.
   - A rewrite must be at least as specific as the original. "MES v12 gives us a different way to handle that folder" is vaguer than the slogan it replaces. Name the thing that does the work ("an agent framework built into the MES reads those files").
   - A rewrite must not invent facts. If a plain version needs a detail the post does not give, write the rewrite with a placeholder and add "(confirm with author)".
   - A rewrite must not create a new slip, such as repeating the phrase from the sentence before it.
6. Count. If more than about a third of the paragraphs are flagged, say so in one line at the top: the draft needs a rewrite pass, not spot fixes. Still give the full table.

## Output

Reply with:

1. One or two sentences: how strong the accent is, which two or three traps dominate, and which sections carry most of it.
2. A table in post order:

| # | As written | Trap | Suggested change |
| --- | --- | --- | --- |
| 1 | "One Agent Framework, Many Problems: **Taming** Multi-Format File Integrations" | Metaphor, slogan title | "Reading CMM, CSV and XML Inspection Files into One Data Collection with a CM MES v12 Agent" |

3. After the table rows, in this order: rows for whole-post patterns, one row for slips (low priority), and one row for spelling that lists the words, the US form, and any occurrences that must stay because they are sample data.

Do not edit the file unless the user asks you to apply the changes. When they do, apply only the rows they accept, with `Edit`, and leave everything else as it is.
