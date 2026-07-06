<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [RFC-MAG-0002: Model C structural impact](#rfc-mag-0002-model-c-structural-impact)
- [Model C: what changes in Magpie's structure](#model-c-what-changes-in-magpies-structure)
  - [Summary of what needs to exist](#summary-of-what-needs-to-exist)
  - [New structure of the Magpie repository (source)](#new-structure-of-the-magpie-repository-source)
    - [Highlighted changes](#highlighted-changes)
  - [New structure of the adopter repository](#new-structure-of-the-adopter-repository)
    - [Differences from the current state](#differences-from-the-current-state)
  - [Schema of the new files](#schema-of-the-new-files)
    - [Skill manifest (in each skill)](#skill-manifest-in-each-skill)
    - [Capability taxonomy](#capability-taxonomy)
    - [Adopter intent](#adopter-intent)
    - [Adopter lock](#adopter-lock)
  - [What the reconciler does, step by step](#what-the-reconciler-does-step-by-step)
  - [Changes to existing skills](#changes-to-existing-skills)
  - [Changes to per-adopter templates](#changes-to-per-adopter-templates)
  - [Changes to the docs](#changes-to-the-docs)
  - [Proposed migration sequence](#proposed-migration-sequence)
  - [Risks and mitigations](#risks-and-mitigations)
  - [What does **not** change](#what-does-not-change)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- SPDX-License-Identifier: Apache-2.0 -->

# RFC-MAG-0002: Model C structural impact

| Field | Value |
|---|---|
| Status | Proposed (under discussion on dev@) |
| Authors | André Ahlert, Apache Magpie working group |
| Tracking issue | [#1](https://github.com/andreahlert/magpie/issues/1) |
| Depends on | [RFC-MAG-0001](RFC-MAG-0001-adoption-models.md) |

# Model C: what changes in Magpie's structure

Details the structural and organizational impact of adopting **Model C (intent + lock)** described in [RFC-MAG-0001](RFC-MAG-0001-adoption-models.md). Covers both the Magpie source repository and the adopter repository.

A linear reading assumes familiarity with the previous RFC.

## Summary of what needs to exist

List of the new pieces Model C requires. Detailed in the sections that follow.

1. **Skill manifest** that is machine-readable in each skill, with structured tags.
2. **Capability taxonomy** that is versioned and canonical (domains, audiences, risk tiers, integrations).
3. **Reconciler** that cross-references `intent.yaml` with the taxonomy and manifests, emitting a lock.
4. **Formal schema** for the `intent.yaml` and `lock` files.
5. **Plan/apply CLI** that drives the declarative cycle.
6. **Override system** that is structured, with defined types (exclude, force-include, pin, param-override).
7. **Migration registry** that documents skill renames, splits and merges across versions.
8. **Parameterizable templates** instead of static files.
9. **Skill registry index** that is aggregated, generated from the manifests.

## New structure of the Magpie repository (source)

Direct comparison with the current layout.

```text
magpie/
├── README.md
├── LICENSE NOTICE .asf.yaml pyproject.toml uv.lock
│
├── agent/
│   ├── mission.md
│   ├── contract.md
│   ├── policies/
│   │   ├── security-model.md
│   │   ├── privacy.md
│   │   └── scope.md
│   ├── prompts/
│   └── taxonomy/                           # NEW
│       ├── domains.yaml
│       ├── audiences.yaml
│       ├── risk-tiers.yaml
│       └── integrations.yaml
│
├── skills/                                 # was .claude/skills
│   ├── security-issue-import/
│   │   ├── manifest.yaml                   # NEW, machine-readable
│   │   ├── SKILL.md                        # prose, as today
│   │   ├── templates/                      # parameterizable files
│   │   │   └── canned-replies.md.j2
│   │   └── params.schema.json              # NEW, accepted params
│   ├── pr-management-triage/
│   │   └── ...
│   └── ...
│
├── runtime/                                # was tools/* services
│   ├── github/ jira/ gmail/ ponymail/
│   └── ...
│
├── workflows/                              # was tools/* orchestrators
│   ├── pr-management/ issue-management/
│   └── ...
│
├── reconciler/                             # NEW
│   ├── resolve.py                          # intent + taxonomy + manifests -> lock
│   ├── plan.py                             # diff between current and new lock
│   ├── apply.py                            # materializes the adopter workspace
│   ├── migrations/                         # renames, splits
│   │   ├── 2026-04-01-split-pr-triage.yaml
│   │   └── 2026-05-15-rename-security-import.yaml
│   └── schemas/
│       ├── intent.schema.json
│       └── lock.schema.json
│
├── registry/                               # NEW, generated
│   ├── skills-index.json                   # build artifact, aggregated from manifests
│   └── capabilities-matrix.md              # generated, human-readable
│
├── evals/
├── data-sources/
│
├── projects/                               # canonical examples
│   └── _example-airflow/
│       ├── .apache-magpie.intent.yaml
│       └── .apache-magpie.lock
│
└── docs/
    ├── architecture/
    ├── governance/
    ├── rfcs/
    ├── setup/
    ├── adoption/                           # NEW, adopter guide
    │   ├── intent-cookbook.md
    │   ├── override-guide.md
    │   ├── plan-apply-cycle.md
    │   └── migration-when-skill-renames.md
    └── contributing.md
```

### Highlighted changes

- `projects/_template/` disappears as a folder of 20 static .md files. It becomes `projects/_example-airflow/` with real intent + lock. The actual templates live inside each skill in `skills/<n>/templates/`.
- `.claude/skills/` becomes `skills/` at the top level. A skill stops being a Claude packaging detail and becomes a first-class citizen of the framework.
- `reconciler/` is the engine. Without it the model does not exist.
- `registry/` is a build artifact (committed or regenerated in CI, a process choice).
- `agent/taxonomy/` is the canonical vocabulary. Any skill that invents a tag outside it fails the validator.

## New structure of the adopter repository

Today the adopter gets symlinks inside `.claude/skills/`, a partial lock (install pin only), and a loose overrides directory. With Model C:

```text
<adopter-repo>/
├── .apache-magpie.intent.yaml             # SOURCE OF TRUTH, committed
├── .apache-magpie.lock                    # generated by `magpie apply`, committed
├── .apache-magpie.local.lock              # gitignored, "what was fetched"
├── .apache-magpie/                        # gitignored, framework snapshot
├── .apache-magpie-overrides/              # structured, committed
│   ├── canned-replies/
│   │   └── pr-management-triage.md         # one-off template override
│   └── params/
│       └── security-issue-import.yaml      # override of declared params
└── .claude/skills/                         # symlinks generated by apply
    └── ...
```

### Differences from the current state

- The lock becomes a complete capability contract, not just an install pin.
- The override stops being a free-form folder and gains typed subfolders. A template override goes in `canned-replies/`, a parameter override in `params/`. This lets the reconciler validate.
- Symlinks are the output of `apply`, not a manual choice during onboarding.

## Schema of the new files

### Skill manifest (in each skill)

```yaml
# skills/security-issue-import/manifest.yaml
id: security-issue-import
version: 1.4.2
domains: [security]
audiences: [maintainer-inbound]
risk-tier: suggest-only
integrations: [github, ponymail, vulnogram]
requires: [setup-isolated-setup-install]
templates:
  - canned-replies.md.j2
params:
  schema: params.schema.json
  defaults: params.defaults.yaml
status: stable
```

### Capability taxonomy

```yaml
# agent/taxonomy/risk-tiers.yaml
tiers:
  - id: suggest-only
    order: 1
    description: "Agent reads and proposes. Human acts."
  - id: draft-pr
    order: 2
    description: "Agent writes PR. Human reviews and merges."
  - id: write-comment
    order: 3
  - id: write-merge
    order: 4
    description: "Auto-merge narrow scope."
```

```yaml
# agent/taxonomy/domains.yaml
domains:
  - id: security
    description: "Security report flow, CVE, embargo."
  - id: pr-queue
  - id: issue-queue
  - id: contributor-lifecycle
  - id: dev-cycle
```

### Adopter intent

```yaml
# .apache-magpie.intent.yaml
framework:
  install: { method: git-branch, ref: main }
capabilities:
  domains: [security, pr-queue]
  audiences: [maintainer-inbound]
  risk-tier-max: draft-pr
  integrations: [github, jira, ponymail, vulnogram]
overrides:
  exclude: [pr-management-code-review]
  force-include: [contributor-nomination]
  pin:
    security-issue-import: "1.4.2"
  params:
    security-issue-import:
      cve-allocator-email: security@adopter.org
```

### Adopter lock

```yaml
# .apache-magpie.lock
generated-from: .apache-magpie.intent.yaml
generated-at: 2026-05-28T14:00:00Z
framework-version: 1.4.0
skills:
  security-issue-import:
    version: 1.4.2
    source: intent.domains
    integrations-resolved: [github, ponymail, vulnogram]
  security-issue-deduplicate:
    version: 1.6.0
    source: intent.domains
  contributor-nomination:
    version: 0.3.1
    source: intent.overrides.force-include
exclusions:
  pr-management-code-review:
    reason: intent.overrides.exclude
checksum: sha256:abc123...
```

## What the reconciler does, step by step

1. **Loads the canonical taxonomy** for the declared framework version.
2. **Validates intent** against the schema. Errors: nonexistent domain, nonexistent risk-tier, an override pointing to a nonexistent skill.
3. **Filters the skill registry** by the intent criteria:
   - Skills whose `domains` intersect `intent.capabilities.domains`.
   - The skill's risk tier is less than or equal to `intent.capabilities.risk-tier-max`.
   - Audiences intersect.
   - The skill's integrations are a subset of those declared.
4. **Applies overrides**:
   - Removes skills in `exclude`.
   - Adds skills in `force-include` (even outside the criteria; emits a warning if it violates risk-tier).
   - Resolves `pin` versus the latest available version.
5. **Resolves dependencies**: each skill's `requires` pulls in auxiliary skills (for example, setup).
6. **Emits a candidate lock**.
7. **Plan**: compares the candidate lock with the current lock. Emits a readable diff.
8. **Apply**: writes the lock, materializes symlinks, writes the resolved templates into the adopter workspace, fires the post-checkout hook.

## Changes to existing skills

Each skill gains:

- `manifest.yaml` with structured tags.
- `params.schema.json` if it accepts adopter parameters.
- `templates/` if it generates files in the adopter workspace (Jinja2 templates or similar).
- `params.defaults.yaml` with safe defaults.

SKILL.md still exists; it is the prose for the human and the agent. The manifest is the contract for the machine.

Cost: each skill today (~17) needs to be audited and gain a manifest. Rough estimate: 30 minutes per skill with a simple manifest, 2 hours for the ones with non-trivial templates.

## Changes to per-adopter templates

Today `projects/_template/` has 20+ static `.md` files. Those files move to live inside the skills that actually consume them, as parameterizable Jinja2 templates:

```text
skills/pr-management-triage/templates/
├── triage-ci-check-map.md.j2
├── triage-comment-templates.md.j2
└── canned-responses.md.j2
```

The adopter parameterizes via `intent.overrides.params` or replaces the whole template via `.apache-magpie-overrides/canned-replies/pr-management-triage.md`.

`projects/_template/` in the Magpie repo becomes `projects/_example-airflow/`: a real intent + lock that serves as a navigable example, not a copy-paste folder.

## Changes to the docs

Documentation gains a new folder `docs/adoption/` with:

- **`intent-cookbook.md`**: examples of intent.yaml per project profile (a small project with triage only, an ASF project with the full security flow, a non-ASF project, and so on).
- **`override-guide.md`**: when to use exclude vs force-include vs pin. Anti-patterns.
- **`plan-apply-cycle.md`**: how to run `magpie plan`, read the diff, apply.
- **`migration-when-skill-renames.md`**: how the reconciler warns that a skill has changed, and what to edit in the intent.

Existing docs that need updating:

- `docs/setup/README.md`: the adoption flow changes. Setup install stays, takeover changes to "edit intent, run plan, run apply".
- `docs/modes.md`: starts explaining modes as a **projection of the intent**, not as internal organization. Still useful as external narrative.
- `README.md` top: replace "skill families" with capabilities language.

## Proposed migration sequence

You cannot swap everything in one PR. The sequence:

1. **PR 1**: introduce `agent/taxonomy/` + schemas. No behavior change.
2. **PR 2**: introduce `manifest.yaml` in one pilot skill (security-issue-import). No reconciler yet.
3. **PR 3**: backfill manifests across the rest of the skills, in waves (one family per PR).
4. **PR 4**: build `registry/skills-index.json` in CI. Manifest validator against the taxonomy.
5. **PR 5**: reconciler in `plan`-only mode, no apply. Adopters can run it to see what would happen.
6. **PR 6**: `apply` in opt-in mode, behind a flag. Pilot supporters try it out.
7. **PR 7**: Jinja2 templates in one pilot skill.
8. **PR 8 onward**: migrate the templates from `projects/_template/` into the skills, deprecate `_template/` in favor of `_example-airflow/`.
9. **Final PR**: flip the default. Setup starts operating in Model C.

Each PR is reversible. The current adopter does not break while the sequence runs.

## Risks and mitigations

| Risk | Mitigation |
|-------|-----------|
| Reconciler becomes a god-component | Keep the rules explicit in YAML/data, code only executes rules. No secret heuristics. |
| Adopter abuses overrides and loses the intent regime | Linter in `plan`: if >5 overrides, suggest rethinking capabilities. Warning, not error. |
| Manifest drifts out of sync with SKILL.md | CI validates: common fields (status, domains) match. PR review checklist requires a synchronized update. |
| Lock schema changes, breaking old adopters | Versioned schema. Reconciler supports N-2 versions. Migration tool offers an upgrade. |
| Skill rename breaks the lock | Migration registry maps old-id to new-id. Reconciler applies an auto-rename in `plan`, the adopter approves. |
| Heavy learning curve for the PMC | `intent-cookbook.md` with ready-made examples. Onboarding via an interactive `magpie init` that generates an initial intent.yaml. |

## What does **not** change

- The modes (Triage, Mentoring, Drafting, Pairing, Auto-merge) survive as external narrative in MISSION.md and as a projection derived from the intent. They do not become a unit of technical configuration.
- The install mechanism (svn-zip, git-tag, git-branch) stays. The lock keeps pinning the install method.
- Sandbox, secure-agent setup, permission rules: orthogonal to the adoption model. Untouched.
- The symlinks in the adopter's `.claude/skills/` remain the way Claude Code sees the skills. Only who decides which symlinks exist has changed.
