# Ondexx Knowledge Bots - Production Setup Runbook

Version 1.0 | October 2026

**Purpose.** How to stand up any Ondexx bot in production, from raw extracts landing in
SharePoint through to a live agent answering questions.

**Operating constraint.** Nothing runs from a laptop. Every production step runs through
SharePoint, GitHub, Power Apps or Power Automate.

---

## 1. What we have built so far

Three bots exist today. All three pass every build and corpus gate.

| Bot | Domain | Agents | Documents | Status |
|---|---|---|---|---|
| **ManuSource** | Life, AD&D, CI and disability claims | 5 | 650 | SME testing agreed, 9/10 evals passing |
| **PMA** | Plan Member Administration | 3 | 634 | In test |
| **SFS** | Sponsor Financial Services | 5 | 498 | Built, scope awaiting SME review |

**13 agents. 10 libraries. 1,782 procedure documents.**

### What already works

- **One builder for all manuals** - `build_studio.ps1`, driven entirely by configuration
- **Automated gates** that block a bad build rather than publishing it
- **Corpus validation** across all 1,782 documents, zero hard failures
- **Evaluation sets** generated per bot, runnable inside Copilot Studio
- **Ondexx source URLs** on every answer, so users can open the original page
- **Taxonomy separated from logic**, ready to become a SharePoint list

### What is not yet in place

- The build still runs on a workstation rather than in GitHub Actions
- The taxonomy is a configuration file, not yet a SharePoint list
- There is no Power Apps front end yet
- Raw intake is not yet a SharePoint site

This runbook describes the target state and how to get there.

---

## 2. The three-site model

```
   SITE 1 - RAW                SITE 2 - STAGING            SITE 3 - CLEARANCE
   raw Ondexx extracts   -->   generated, reviewable  -->  approved content
   from the data team          bot content                 the agents read

        (input)                   (build output)              (published)
```

### Site 1 - Raw

The data team's drop zone. Nothing here is ever edited in place.

| | |
|---|---|
| **Contains** | `.html`, `.docx`, `.pdf` exports straight from Ondexx |
| **Structure** | one folder per manual, then one folder per chapter |
| **Who writes** | data team only |
| **Who reads** | the build pipeline |

Recommended metadata columns: `Manual`, `Source Page ID`, `Source URL`, `Extracted Date`,
`Status`, `Owner`.

> **Rule.** Raw files are append-only. A correction creates a new version or a replacement
> file. Editing in place destroys the ability to reproduce a build.

### Site 2 - Staging

Where the build publishes its output for review. This is what SMEs test against.

| | |
|---|---|
| **Contains** | `.txt` only, chapter folders preserved |
| **Structure** | one library per specialist agent |
| **Who writes** | the build pipeline only |
| **Who reads** | SMEs, reviewers |

### Site 3 - Clearance

The approved set. **This is what Copilot Studio points at.**

| | |
|---|---|
| **Contains** | same structure as Staging |
| **Who writes** | the promotion flow, after sign-off |
| **Who reads** | Copilot Studio agents, end users |

> **Why three and not two.** Evaluations must confirm a build *before* it reaches users.
> That only works if the thing under test is not already live. Clearance also gives you a
> rollback: restore the last known-good Staging snapshot.

---

## 3. Who owns what

This is the most important table in this document.

| Layer | Technology | Owner | Changes how often |
|---|---|---|---|
| Raw intake | SharePoint Site 1 | Data team | per extract |
| **Taxonomy** | **SharePoint list + Power Apps** | **Business** | **weekly** |
| Agent behaviour rules | GitHub (`*-spec.json`) | Technology | rarely |
| Build and gates | GitHub Actions | Technology | rarely |
| Staging content | SharePoint Site 2 | generated | per build |
| Clearance content | SharePoint Site 3 | promotion flow | per release |
| Runtime | Copilot Studio | Technology | per release |

**Business owns what goes where. Technology owns how agents behave.**

---

## 4. The taxonomy - the business control surface

One SharePoint list decides which specialist answers which questions, and therefore which
library each document lands in.

### List columns

| Column | Type | Example |
|---|---|---|
| Subject Area | Text | `LTD` |
| Manual | Choice | `ManuSource` |
| Answered By | Choice / Lookup | `ManuSource Claims` |
| Library Name | Text | `Ondexx - ManuSource - Claims` |
| Documents | Number | `14` |
| Status | Choice | `Active` / `Draft` / `Retired` |
| Business Owner | Person | |
| Notes | Multi-line text | |

### What a change does

```
change "Answered By" on a row
        |
        v
taxonomy updates
        |
        v
build reassigns those documents
        |
        v
they land in a different library
        |
        v
a different specialist answers them
```

No code is written. No engineering request is raised.

> **The `Documents` column matters.** Moving `Commissions` moves 107 documents. The business
> owner should see the weight of a decision before making it.

### Power Apps

A thin canvas app over the list. Its job is to capture decisions, not to transform content.

- Edit `Answered By` and `Status`
- Prevent a subject area being assigned to two agents
- Prevent retiring an agent that still holds content
- Submit for approval
- Raise a build request

**Power Apps must contain no transformation logic.** It writes to SharePoint; GitHub does
the work.

---

## 5. The build pipeline - GitHub Actions

GitHub Actions is the production build engine. The same input always produces the same output.

### What the workflow does

1. Validate the registry JSON parses, and warn on non-ASCII characters
2. Build all agent instructions from configuration
3. Validate all three corpora
4. Verify upload-tree assignment - no chapter assigned twice, none orphaned
5. Self-test the evaluation runner
6. Report instruction headroom, flagging anything under 150 characters free
7. Publish artefacts

**A build that fails a gate produces nothing.** There is no option to publish past a failure.

### The gates

| Gate | What it prevents |
|---|---|
| Character budget <= 8,000 | an instruction the platform would silently truncate |
| Taxonomy cross-check | a subject area with no agent, or with two |
| `MUST search` present | an agent answering from memory instead of documents |
| `Answered by:` contract | an answer you cannot attribute or score |
| No relay rule | the router regressing to a mode that strips citations |
| Starter validity | malformed data reaching the evaluation set |
| Document size and naming | oversize documents split mid-procedure |
| Provenance | a document with no link back to its source page |

### Why the transform stays in GitHub

Splitting a document at heading boundaries is 221 lines of string handling. Expressing that
in Power Automate expressions would be fragile and effectively unmaintainable.

**Power Automate orchestrates. GitHub transforms.** That line should not move.

---

## 6. Power Automate - orchestration only

| Trigger | Action |
|---|---|
| New file in Raw | notify the owner, raise a build request |
| Taxonomy row changed | commit the change, trigger the build |
| Build complete | notify the requester with the result |
| Gates passed | notify reviewers that Staging is ready |
| Approval granted | copy Staging to Clearance |
| Build failed | notify with the gate message, which is specific |

---

## 7. Copilot Studio - runtime only

### Per bot

- One **parent** agent - the router. Holds no knowledge by design.
- One **child** agent per specialist - each with exactly one library, no overlap.

### Connected agent settings - verified working

Apply to **every** connected agent:

| Setting | Value |
|---|---|
| Pass conversation history | **ON** |
| Trigger a conversation start event | **OFF** |
| Completion - After running | **Don't respond (default)** |
| Advanced - Outputs available | **All** |

> **`Outputs available` must be `All`, not `Specific`.** `Specific` with an empty Outputs
> list passes nothing back to the parent, so the orchestration runtime substitutes its own
> message and the answer appears twice. This combination was verified working 2026-10-06.

### System topics - disable on the parent

Disable or customise **Fallback**, **Did not understand** and **Escalate**.

Left enabled they fire before orchestration finishes and swallow the turn. Every refusal
path breaks - the agent returns silence instead of a clean "that is outside what I cover".

**Symptom:** a reply with no `Answered by:` line, or an empty response.

### Access control

Permissions live on the **library**, not the agent. Proven: a user without permission on the
underlying library cannot use the bot. Nothing new to administer.

---

## 8. Quality assurance - three layers

| Layer | What it catches | Can it confirm correctness? |
|---|---|---|
| **Automated gates** | structural faults - size, naming, coverage, attribution | No |
| **Evaluation** | drift in routing and answer quality | No |
| **SME review** | whether the procedure is right and usable | **Yes - only this** |

### What an evaluation pass means

> A pass means **unchanged**, not **correct**.

Evaluations lock in known-good behaviour so you notice drift. Only an SME can confirm the
procedure was right in the first place. Do not let a green evaluation substitute for SME
review.

### Copilot Studio evaluations

- Import format: two columns, `Question` and `Expected response`
- Maximum 100 cases per set, 1,000 characters per question
- `Answer quality` and `Meaning match` work directly from the CSV
- `Tool use` and `Keyword match` need per-case configuration in the portal - they return
  `Invalid` on CSV import alone
- Results retained 89 days; export to CSV to keep them longer

---

## 9. Release sequence

```
1  Raw extract lands in Site 1
2  Business confirms taxonomy rows
3  GitHub Actions builds and gates
4  Artefacts publish to Staging (Site 2)
5  Evaluation runs
6  SME reviews and signs off
7  Promotion flow copies Staging to Clearance (Site 3)
8  Copilot Studio reindexes
```

**Promotion is not the same moment as live.** SharePoint-to-Copilot-Studio reindexing has
latency. Plan for a lag between approving a change and users seeing it.

---

## 10. Operating rules

1. Never hand-edit generated agent instructions - the next build reverts them silently
2. Never flatten the corpus folders on upload - routing tables cite chapters by folder name
3. Never bypass the taxonomy list
4. Never promote on a failed build or failed evaluation
5. Never use a workstation as the production execution engine
6. Never edit a raw extract in place

---

## 11. The one thing that cannot be automated

**Copilot Studio has no API for agent instructions.** A person pastes them.

The corpus pipeline is fully automated. The agent configuration is not. This is a platform
limit, not a gap in the design. If anyone has described this as fully automated, correct it
early rather than late.

Scope of the manual work: roughly 10 minutes per agent one-time, then only when an
instruction actually changes - which means only when someone edits a spec or taxonomy.

---

## 12. Getting from here to there

| # | Step | Owner | Blocks |
|---|---|---|---|
| 1 | Commit the repository and push to GitHub | Technology | everything below |
| 2 | Enable the Actions workflow | Technology | builds leaving the laptop |
| 3 | Create the three SharePoint sites | Platform | intake and promotion |
| 4 | Create the taxonomy list, seeded from the current files | Business + Platform | Power Apps |
| 5 | Build the Power Apps front end | Platform | business self-service |
| 6 | Request an app registration with `Sites.ReadWrite.All` | Governance | scripted upload |
| 7 | Wire the Power Automate flows | Platform | end-to-end automation |

> **Step 6 is worth starting now.** Without it, uploads are browser drag-and-drop. The
> `az` CLI token carries no SharePoint scope, so no script can do it today.

---

## Appendix - the minimal production stack

| Layer | Technology |
|---|---|
| Raw intake, content, taxonomy, access control | **SharePoint** |
| Business editing and approval | **Power Apps** |
| Triggers and orchestration | **Power Automate** |
| Source control, build, gates, audit trail | **GitHub + Actions** |
| Conversation and routing | **Copilot Studio** |

No servers. No database. No search index. Nothing to patch, scale or fail over.

If the build pipeline stopped tomorrow, the bots keep answering.
