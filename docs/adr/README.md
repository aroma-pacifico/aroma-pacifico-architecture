# Architecture Decision Records

This directory holds the Architecture Decision Records (ADRs) of the Aroma Pacífico pilot. An ADR records one significant structural choice, technology selection or governance boundary, the alternatives that were weighed, and the consequences the team accepts by taking it. ADRs exist so that every choice can be justified in the final defense before the Federation architect, and so that no decision stays buried as transient prose inside a design document (`standards/documentation.md` section 5).

The master index of ADRs lives in the [decision log](../decision-log.md#architecture-decision-records), next to the assumptions (`ASM-NNN`) and pending decisions (`PD-NNN`) that ADRs often depend on. The [case statement](../case/case-statement.pdf) is the only source of requirements; an ADR may decide how a requirement is met, but it never adds a requirement.

## Lifecycle

An ADR moves through three statuses. The status lives both in the ADR file header and in the decision log index, and the two must match.

| Status | Meaning | Who moves it here |
|---|---|---|
| Proposed | The ADR is written and open for review. The decision is not yet binding. | The ADR author, when the pull request that adds the file is opened. |
| Accepted | The lead approved the ADR. The decision is binding for all later work. | The lead, by approving and merging the pull request. |
| Superseded | A later ADR replaced this decision. The file stays in place and is never deleted; its header names the superseding ADR. | The author of the superseding ADR, in the same pull request. |

Before an ADR file exists, the decision log index may carry the interim status `Decided by owner, ADR pending`. It means the owner has taken the decision and the ADR task still has to write it down, weigh the alternatives and move it to Proposed.

An Accepted ADR is not edited to change its decision. To change a decision, write a new ADR that supersedes it, set the old one to Superseded and update both rows in the decision log. Typo and link fixes that do not alter the decision are allowed.

## Writing an ADR

The procedure below is sequential, so it is a numbered list.

1. Take the next free `ADR-NNN` from the decision log index, or use the ID already reserved there.
2. Copy [`template.md`](template.md) to `adr-NNN-<kebab-slug>.md` in this directory.
3. Fill every section. Quote the case statement for each requirement the decision answers, and register any value the case statement does not give as an `ASM-NNN` entry in the decision log.
4. Set the status to Proposed in the file and in the decision log index, and link the file from the index row.
5. Open a pull request that references the roadmap issue as `aroma-pacifico/aroma-pacifico-roadmap#<n>`.

## Files

| File | Purpose |
|---|---|
| `README.md` | This lifecycle guide. |
| [`template.md`](template.md) | The section skeleton every ADR copies. |
| `adr-NNN-<kebab-slug>.md` | One file per ADR. None is written yet; the reserved file names are listed in the decision log. |
