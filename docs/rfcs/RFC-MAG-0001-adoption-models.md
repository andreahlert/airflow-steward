<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [RFC-MAG-0001: Adoption models (intent vs skill vs hybrid)](#rfc-mag-0001-adoption-models-intent-vs-skill-vs-hybrid)
- [Two models for configuring Magpie in an adopter project](#two-models-for-configuring-magpie-in-an-adopter-project)
  - [The problem](#the-problem)
  - [Model A: Intent-based (Terraform style)](#model-a-intent-based-terraform-style)
    - [The idea](#the-idea)
    - [How the adopter declares](#how-the-adopter-declares)
    - [What Magpie does](#what-magpie-does)
    - [Layman analogy](#layman-analogy)
    - [Advantages](#advantages)
    - [Disadvantages](#disadvantages)
  - [Model B: Skill-based (Ansible style)](#model-b-skill-based-ansible-style)
    - [The idea](#the-idea-1)
    - [How the adopter declares](#how-the-adopter-declares-1)
    - [What Magpie does](#what-magpie-does-1)
    - [Layman analogy](#layman-analogy-1)
    - [Advantages](#advantages-1)
    - [Disadvantages](#disadvantages-1)
  - [Model C: Intent with skill lock (hybrid)](#model-c-intent-with-skill-lock-hybrid)
    - [The idea](#the-idea-2)
    - [How the adopter declares](#how-the-adopter-declares-2)
    - [How the cycle works](#how-the-cycle-works)
    - [Layman analogy](#layman-analogy-2)
    - [Advantages](#advantages-2)
    - [Disadvantages](#disadvantages-2)
    - [When it makes sense](#when-it-makes-sense)
  - [Side-by-side comparison](#side-by-side-comparison)
  - [Where Magpie is today](#where-magpie-is-today)
  - [The design question](#the-design-question)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- SPDX-License-Identifier: Apache-2.0 -->

# RFC-MAG-0001: Adoption models (intent vs skill vs hybrid)

| Field | Value |
|---|---|
| Status | Proposed (Model C recommended; under discussion on dev@) |
| Authors | André Ahlert, Apache Magpie working group |
| Tracking issue | [#1](https://github.com/andreahlert/magpie/issues/1) |
| Related RFC | [RFC-MAG-0002](RFC-MAG-0002-model-c-structural-impact.md) |

# Two models for configuring Magpie in an adopter project

Exploratory document. Explains three possible ways for an open-source project to tell Magpie what it wants it to do. Language is kept accessible for a developer who has never used Terraform or Ansible.

## The problem

Magpie is a platform of skills (agent capabilities). There are skills for issue triage, for reviewing PRs, for handling the security flow, for mentoring a new contributor, and so on. Total: dozens of skills.

A project that wants to adopt Magpie (Apache Airflow, for example) does not want to turn everything on. It wants to choose. The question is: **how does the adopter choose?**

There are two classic ways to model this choice. The names come from infrastructure tooling (Terraform and Ansible), but the concept applies to any system that needs configuration.

## Model A: Intent-based (Terraform style)

### The idea

The adopter describes **what it wants**, not **how to do it**. The system figures out which skills to enable from that.

### How the adopter declares

```yaml
# .apache-steward.lock
capabilities:
  domains: [security, pr-queue]          # areas I care about
  audience: [maintainer-inbound]         # who my audience is
  risk-tier-max: draft-pr                # how far I am willing to go
  integrations: [github, jira, ponymail] # tools I use
```

### What Magpie does

A component called the **reconciler** reads this file and reasons:

- "This project handles security and the PR queue."
- "Max risk is draft-pr (agent writes, human merges). Auto-merge skills are out."
- "Audience is maintainer inbound. Dev-side pairing skills are out."
- "Uses GitHub, Jira, Ponymail. Skills that depend on Gmail are out."

From that intersection, the reconciler **generates** the list of enabled skills and materializes the files in the project.

### Layman analogy

You tell Uber: "I want to go from the airport home, I prefer a car, up to R$50." The app picks the driver and the route. You never asked for "Honda Civic, plate ABC1234, via the Marginal."

### Advantages

- The adopter reasons in **concepts from its own domain** (domains, risk, audience), not in skill names.
- When Magpie adds a new PR-queue skill, it shows up automatically in projects that declared `domains: [pr-queue]`. No manual config edit needed.
- The lock file's language is stable. Internal skills get renamed without breaking anything.
- The conversation with the PMC stays clear: "how much risk do we accept?" is a human decision, not technical trivia.

### Disadvantages

- The reconciler has to exist. Someone writes and maintains it. It carries non-trivial logic.
- Each skill needs **machine-readable metadata** declaring its tags: `domain`, `audience`, `risk-tier`, `integrations`. Today a skill is prose in SKILL.md. It needs structuring.
- The adopter loses granular control. If it wants to enable exactly skill X but not Y from the same domain, it gets uncomfortable. It needs a fine-grained override mechanism.
- Surprise on upgrade: a new skill turns itself on. Good for fast adoption, bad for a conservative project.

## Model B: Skill-based (Ansible style)

### The idea

The adopter **lists the skills it wants** one by one. There is no inference. The tags `(domain, audience, risk)` exist only to help navigate and filter in the documentation.

### How the adopter declares

```yaml
# .apache-steward.lock
skills:
  - security-issue-import
  - security-issue-deduplicate
  - security-cve-allocate
  - pr-management-triage
  - pr-management-code-review
  # auto-merge, mentoring, pairing: not listed, stay out
```

### What Magpie does

Reads the list. Enables exactly what is there. Done.

### Layman analogy

You go to a restaurant and order from the menu: "Caesar salad, margherita pizza, orange juice." The waiter brings exactly those three items. They do not interpret "I am moderately hungry and I like Italian" to choose for you.

### Advantages

- Simple to implement. No reconciler, no capabilities schema, no inference.
- The adopter has full, predictable control. Enabled skill X, gets skill X.
- A Magpie upgrade never enables anything new without the adopter asking. New skills stay dormant until someone edits the lock.
- Skill metadata can stay as prose. Tags are just for documentation.

### Disadvantages

- The adopter has to know the skill catalog. Higher onboarding cost.
- The discussion with the PMC becomes too granular. "Do we enable security-issue-deduplicate?" is a technical question, not a strategic one.
- Related skills have to be enabled together by hand. Forgetting one breaks the flow.
- When Magpie renames or splits a skill, **every adopter breaks**. The lock file references an internal identifier.
- No "max risk" semantics. The adopter can enable auto-merge by mistake if it does not know the taxonomy.

## Model C: Intent with skill lock (hybrid)

### The idea

The adopter declares intent (capabilities). The reconciler resolves the skill list. But the result of that resolution is **written to a lock file**, and the adopter can annotate surgical exceptions in it: pin a skill to a version, exclude a skill the reconciler enabled, force-include a skill the reconciler did not pick up.

It is the marriage of the two previous worlds. Intent guides the common case, the lock covers the special case.

### How the adopter declares

Two files, distinct roles. One is desire, the other is frozen resolution.

```yaml
# .apache-steward.intent.yaml (committed, hand-edited)
capabilities:
  domains: [security, pr-queue]
  audience: [maintainer-inbound]
  risk-tier-max: draft-pr
  integrations: [github, jira, ponymail]

overrides:
  exclude:
    - pr-management-code-review     # we do not want this one, even though it falls in the domain
  force-include:
    - contributor-nomination        # we want this one, outside the declared domain
  pin:
    security-issue-import: "1.4.2"  # locked to this version until we validate the next one
```

```yaml
# .apache-steward.lock (committed, generated by the reconciler)
generated-from: .apache-steward.intent.yaml
generated-at: 2026-05-28T14:00:00Z
skills:
  security-issue-import: { version: "1.4.2", source: intent.domains }
  security-issue-deduplicate: { version: "1.6.0", source: intent.domains }
  pr-management-triage: { version: "2.1.0", source: intent.domains }
  contributor-nomination: { version: "0.3.1", source: intent.overrides.force-include }
  # pr-management-code-review: absent, source: intent.overrides.exclude
```

### How the cycle works

1. The adopter edits `intent.yaml`. Runs `magpie plan`.
2. Plan shows the diff: "will add X, remove Y, keep Z at the pinned version."
3. The adopter runs `magpie apply`. The reconciler writes a new `lock` and materializes the workspace.
4. Both files are committed. The reviewer sees both the **desire** and the **result**.

### Layman analogy

`package.json` versus `package-lock.json` in Node. You write `"react": "^18.0.0"` (intent: I accept any 18.x). npm resolves it to exactly `18.2.0` and freezes it in the lock. If you want to pin a different exact version, you edit the lock or change the range in `package.json`. Both files go to git.

### Advantages

- The common case (90% of adoptions) uses only intent, without touching an individual skill.
- Special cases fit without becoming a broken exception. Override is part of the model, not a hack.
- The lock gives **exact reproducibility**. Two clones of the adopter repo on different machines resolve to the same list.
- A Magpie upgrade runs `magpie plan` before touching anything. The PMC sees the diff and decides. No surprises.
- A PR reviewer in the adopter repo sees **intent + lock + materialized diff** in a single commit. Auditable.
- Renamed skills produce a clear reconciliation error: "skill X (referenced in overrides.pin) no longer exists, migrated to Y." The adopter updates intent, not code.

### Disadvantages

- More files. More concepts to explain (intent vs lock, plan vs apply).
- The reconciler has to exist and be solid. Same cost as Model A.
- Too many overrides and the adopter effectively leaves the intent regime. It becomes skill-based with decoration. Requires discipline and linting ("if you have >5 overrides, rethink the intent").
- A merge conflict in `lock` can be ugly when two PRs change intent at the same time. The solution is known (regenerate), but it needs documentation.

### When it makes sense

It makes sense when you expect adopters with very different profiles: some just want to turn it on and use it (pure intent is enough), others will need a one-off exception without flipping the whole game. Apache works like this. Airflow operates differently from Kafka, both differently from some non-ASF project.

By Magpie's design (the mission speaks of "project autonomy" as a structural starting point), Model C is the one that best preserves that autonomy without falling into the cognitive overload of pure skill-based.

## Side-by-side comparison

| Criterion | A. Pure intent | B. Pure skill | C. Intent + lock |
|----------|----------------|---------------|------------------|
| Unit of choice | Capability | Individual skill | Capability, with per-skill override |
| Who decides the final set | Reconciler | Adopter | Reconciler + adopter overrides |
| Onboarding | Answer 4 questions | Read catalog, choose N | Answer 4 questions, override later if needed |
| Magpie upgrade | New skill turns itself on | New skill stays dormant | `plan` shows diff, adopter decides |
| Breaks on rename | No | Yes, breaks the lock | Clear reconciliation error, guided migration |
| Granular control | Needs a hack | Native | Native via overrides |
| Discussion with PMC | Strategic | Technical | Strategic in the common case, technical in the special case |
| Reproducibility across machines | Good | Exact | Exact |
| Cost to implement | Reconciler + metadata | Documented catalog | Reconciler + metadata + plan/apply |
| Main risk | Surprise on upgrade | Heavy onboarding, drift | Adopter abuses overrides and loses the regime |

## Where Magpie is today

A hybrid leaning toward skill-based, but coarse. The current setup asks about **skill families** (`security`, `pr-management`), which is halfway there:

- Coarser than an individual skill.
- Coarser than a capability.
- No machine-readable metadata.
- No reconciler.

The lock file today stores only the **install pin** (where the framework came from, which version). It does not store capability config or a skill enable list. The choices live in the symlinks created during takeover, which is opaque state.

## The design question

Three options, not two. The default defines the platform's face.

- **A. Pure intent:** "tell me what you need, I assemble it." Adoption at scale, but surprise on upgrade and override sits outside the model.
- **B. Pure skill:** "open catalog, build your kit." Maximum control, but heavy onboarding and breakage on rename.
- **C. Intent + lock (hybrid):** intent as the main conversation, lock as the contract, override as a legitimate exception. More concepts, better preserves project autonomy.

C is the most aligned with what MISSION.md declares about project autonomy. It costs more to implement (reconciler + plan/apply + structured skill metadata), and it demands discipline against override abuse.

The choice is not only technical. It determines the kind of conversation an adopter has with Magpie at adoption time, and the kind of upgrade it receives over time.
