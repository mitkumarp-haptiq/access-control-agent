# Access Control Documentation Agent

A scheduled agent that keeps Confluence "Access Control" documentation in sync with
Jira access-request tickets, for **audit traceability**.

It is a **generic, config-driven engine**: the workflow lives in [`SKILL.md`](SKILL.md),
and every team-specific identifier is supplied at run time through the
`ACCESS_SYNC_CONFIG` environment variable — so one engine serves any team without editing
the skill.

## What it does

Once per day it:

1. Reads Jira tickets in the configured project labeled as a **grant** (`Access`) or a
   **removal** (`accessRevoke`), skipping tickets already completed
   (`accessAdded`/`accessRevoked`) or still in **Backlog**.
2. Parses **every** beneficiary named on the ticket (one ticket may list several people) —
   from the description, structured fields, and comments, not just the summary — and
   resolves each to a stable Jira **accountId** (never the assignee, who is the engineer
   fulfilling the request).
3. For each person, finds or creates their page under the Confluence **Access Control**
   parent page and updates a standardized per-tool access table (Access Added for grants,
   Access Removed for removals).
4. Once a ticket is **Done** and every beneficiary's page is updated, swaps its label
   (`Access`→`accessAdded`, or `accessRevoke`→`accessRevoked`) so it isn't reprocessed.

The full, authoritative procedure lives in [`SKILL.md`](SKILL.md); this section is only a
summary.

## Design principles

The agent is built to be **deterministic** and audit-friendly:

- **Identity by accountId, not name.** People are resolved to a canonical Jira accountId;
  pages are matched by the canonical `displayName` that accountId resolves to. Ambiguous
  names (e.g. a first name + last initial) are disambiguated against active users before
  ever being flagged. A departed employee on a removal ticket, already deleted from Jira,
  is matched by name against an existing Access Control page instead.
- **Per-person, independent.** Each beneficiary on a ticket is processed on its own; one
  person failing to resolve doesn't block the others, and the ticket's label is only
  advanced once every person on it is done.
- **Full table re-render.** The per-person table is regenerated from a data model on every
  run rather than surgically edited, which keeps `rowspan`/cell-count structure consistent.
- **Config or nothing.** Team identifiers come from `ACCESS_SYNC_CONFIG`. If it is missing
  or invalid, the agent stops and reports — it never falls back to another team's values.
- **Fail safe, not silent.** On a genuine ambiguity or write failure the agent comments on
  the ticket tagging a reviewer, and makes no page/label changes for that person.

## Configuration

All team-specific values are read from a single JSON environment variable,
`ACCESS_SYNC_CONFIG`. The workflow, tool mapping, and table format are **not** config —
they live in `SKILL.md`.

**Required** (the agent refuses to run if any is missing):

| Key | What it is |
| --- | --- |
| `site` | Atlassian site host, e.g. `yourco.atlassian.net` |
| `cloudId` | Atlassian cloud site id (passed on every MCP call) |
| `jira.projectKey` | Jira project holding the access tickets |
| `confluence.spaceKey` | Confluence space key |
| `confluence.parentPageId` | The "Access Control" parent page id |
| `reviewerAccountId` | accountId to @mention on errors |

**Optional** (defaults shown):

| Key | Default |
| --- | --- |
| `labels.grant` / `labels.removal` | `Access` / `accessRevoke` |
| `labels.grantDone` / `labels.removalDone` | `accessAdded` / `accessRevoked` |
| `skipStatuses` | `["Backlog"]` |
| `confluence.referencePageUrl` | none (an existing person page to mirror) |

A complete, copy-pasteable reference lives in
[`config.example.json`](config.example.json) — it uses **dummy** placeholders that match
the real value shapes (`projectKey` `XXX`, `spaceKey` `XX1`, `parentPageId`
`0000000000`, `reviewerAccountId` `000000:00000000-…`). Copy it, replace every dummy
with your team's real values, and save as your team's config file (e.g. `config.json`).

**Critical — set `ACCESS_SYNC_CONFIG` as one line.** The Claude Code routine env field
often truncates multi-line JSON to just `{`, which is invalid and stops the skill.
Do not paste the pretty-printed file, and do not wrap the value in quotes.

Easiest way (macOS) — minify and copy to the clipboard, then paste into the routine:

1. Put your filled-in team config in a file (e.g. `config.json`) in this repo directory.
2. In a terminal, from that directory, run:

```bash
jq -c . config.json | pbcopy
```

3. That command copies a single-line JSON string to your clipboard. Open the routine →
   environment variable `ACCESS_SYNC_CONFIG` → paste (⌘V). Do not add quotes around it.
4. Save the routine, then confirm the stored value is the full JSON (not just `{`).

Requires [`jq`](https://jqlang.github.io/jq/) (`brew install jq` if needed). `pbcopy` is
built into macOS.

### Where to find each value

Use the steps below to replace the dummies in [`config.example.json`](config.example.json).
Haptiq is shown as a worked example; other teams use the same URL patterns on their site.

#### Site-wide (Atlassian)

| Key | Example shape in config | How to find it |
| --- | --- | --- |
| `site` | `haptiq.atlassian.net` | Host from any Atlassian URL → `haptiq.atlassian.net`. |
| `cloudId` | `"..."` | Open (logged in): `https://haptiq.atlassian.net/_edge/tenant_info` → JSON `"cloudId"`. |

#### From Confluence

Start from the team's **Access Control** parent page. Example URL shape (dummies — replace with yours):

`https://haptiq.atlassian.net/wiki/spaces/XX1/pages/0000000000/Access+Control`

| Key | Example shape in config | How to find it |
| --- | --- | --- |
| `confluence.spaceKey` | `XX1` | From `/wiki/spaces/<SPACE_KEY>/...` → `XX1`. |
| `confluence.parentPageId` | `0000000000` | From `/pages/<PAGE_ID>/...` → `0000000000`. |
| `confluence.referencePageUrl` | optional URL under that space | Optional. Open an existing well-formed person page under that Access Control parent and copy its full URL — the agent mirrors that page's structure for new pages. |

#### From Jira

| Key | Example shape in config | How to find it |
| --- | --- | --- |
| `jira.projectKey` | `XXX` | Access-request project key — from any issue (`XXX-123` → `XXX`), project settings, or the project URL. |
| `reviewerAccountId` | `000000:00000000-0000-0000-0000-000000000000` | Person to @mention on sync errors. Open (logged in): `https://haptiq.atlassian.net/rest/api/3/user/search?query=NAME_OR_EMAIL` — copy `"accountId"` (same `number:uuid` shape). For yourself: `https://haptiq.atlassian.net/rest/api/3/myself`. |
| `labels.*` / `skipStatuses` | Haptiq defaults (real) | Labels/statuses on those Jira tickets. Defaults match Haptiq (`Access`, `accessRevoke`, `accessAdded`, `accessRevoked`; skip `Backlog`). Change only if your project differs. |

### Tool mapping lives in the skill, not config

The **TOOL MAPPING**, the standard row set, and the table `FORMAT` are written directly in
`SKILL.md`. If your team's tool stack differs (different systems or environments), edit
those sections of `SKILL.md` rather than passing them through the environment.

## Contents

| File | Purpose |
| --- | --- |
| `SKILL.md` | The generic engine: frontmatter + step-by-step workflow, tool mapping, and table format. |
| `config.example.json` | Reference `ACCESS_SYNC_CONFIG` with dummy IDs (same shapes as real) — copy, replace, supply as the env var. |

## Usage (Claude Code cloud routine)

This skill is meant to run as a **Claude Code cloud routine** — one routine per team —
on a daily schedule. Each run is a fresh cloud session: it loads `SKILL.md` from this
repo and follows that workflow end to end. Team-specific values come only from
`ACCESS_SYNC_CONFIG` in the routine's cloud environment.

### Per-team setup

Create a routine at [claude.ai/code/routines](https://claude.ai/code/routines) (or via
`/schedule` in the CLI):

1. **Repository** — Add this repo so the session can read `SKILL.md` at the repo root.
2. **Environment variable** — Set `ACCESS_SYNC_CONFIG` using the one-line copy/paste
   steps under **Configuration** above (`jq -c . config.json | pbcopy`, then paste).
   Multi-line paste is often truncated to `{` and the run aborts.
3. **Connectors** — Include Jira and Confluence. The routine runs under the account that
   owns it (no approval prompts during the run), so that account needs issue
   read/label-edit/comment and Confluence page search/create/update.
4. **Schedule** — Daily (cadence is routine metadata, not part of `SKILL.md`).
5. **Instructions** — Use a prompt like the one below (swap the site and repo URL for
   your team).

### Routine instructions (prompt)

```text
Run the daily Jira → Confluence "Access Control" sync for <your-site>.atlassian.net.

Get the workflow from its source of truth and follow it exactly:
1. Clone (or pull, if already cloned) https://github.com/<org>/access-control-agent.git
2. Read SKILL.md at the repo root and execute it end to end.
Do not improvise beyond what SKILL.md specifies.
```

Example for Haptiq:

```text
Run the daily Jira → Confluence "Access Control" sync for haptiq.atlassian.net.

Get the workflow from its source of truth and follow it exactly:
1. Clone (or pull, if already cloned) https://github.com/mitkumarp-haptiq/access-control-agent.git
2. Read SKILL.md at the repo root and execute it end to end.
Do not improvise beyond what SKILL.md specifies.
```

Keep this repo as the shared source of truth: each team runs its own routine against the
same `SKILL.md`, with its own `ACCESS_SYNC_CONFIG`, and improvements land in one place.
