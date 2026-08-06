---
name: jira-confluence-access-sync
description: Config-driven daily sync of Jira access-grant/removal tickets into per-person Confluence "Access Control" pages; team-specific IDs come from the ACCESS_SYNC_CONFIG environment variable, while the workflow, tool mapping, and table format live in this skill
---

Sync Jira tickets into per-person Confluence "Access Control" pages. Two ticket types: GRANTS (label `{labels.grant}`) and REMOVALS (label `{labels.removal}`). Team-specific identifiers (site, Jira project, Confluence space/page, reviewer, labels) come from configuration; the workflow, TOOL MAPPING, FORMAT, and table rules below are part of this skill and apply to every run.

PRINCIPLES: Identify people by stable Jira accountId (never ticket display text, never the Assignee). Match pages by the canonical displayName that accountId resolves to. Map tools via the TOOL MAPPING below. Always REGENERATE the whole table from a data model — never edit cells by hand. A ticket may name MORE THAN ONE person — process each independently. This is a self-contained daily task with no memory of prior runs; rediscover current state from Jira/Confluence each time.

0. LOAD CONFIG (do this first, before anything else):
   - Read the environment variable `ACCESS_SYNC_CONFIG` (e.g. `printenv ACCESS_SYNC_CONFIG`) and parse it as JSON.
   - REQUIRED keys — if any is missing/empty, or the variable is unset/empty/invalid JSON, STOP immediately and report the problem. Do NOT improvise, do NOT fall back to any default or previously-known values, do NOT touch Jira or Confluence:
     * `site` (e.g. "your-org.atlassian.net")
     * `cloudId` (Atlassian cloud site id — pass on every MCP call; do NOT call `getAccessibleAtlassianResources` to look it up)
     * `jira.projectKey`
     * `confluence.spaceKey`, `confluence.parentPageId`
     * `reviewerAccountId`
   - OPTIONAL keys — apply these defaults when absent:
     * `labels.grant` = "Access", `labels.removal` = "accessRevoke", `labels.grantDone` = "accessAdded", `labels.removalDone` = "accessRevoked"
     * `skipStatuses` = ["Backlog"]
     * `confluence.referencePageUrl` = none
   - Throughout the steps below, `{...}` means "the value from config" — e.g. `{jira.projectKey}`, `{labels.grant}`. Pass `{cloudId}` on every Atlassian MCP tool call. TICKET_URL for a ticket KEY is `https://{site}/browse/{KEY}`.

1. READ tickets: JQL `project = {jira.projectKey} AND labels IN ("{labels.grant}","{labels.removal}") ORDER BY updated ASC`. Label `{labels.grant}` = GRANT; `{labels.removal}` = REMOVAL (a removal may carry only the removal label).

2. SKIP the ticket entirely if it has label `{labels.grantDone}` or `{labels.removalDone}` (already done), or if its status name is in `{skipStatuses}` (leave labels/pages untouched so a later run revisits it).

3. PARSE ALL BENEFICIARIES (there may be several on one ticket):
   - ALWAYS read the DESCRIPTION — it is the primary source and must not be skipped. Also check structured fields (Beneficiary/Requested for/User) and comments. Do NOT rely on the summary alone.
   - Extract EVERY distinct person named as a beneficiary (a ticket often lists multiple people, and may name multiple tools/envs per person). Never the Assignee (the engineer fulfilling it).
   - If no beneficiary can be found anywhere on the ticket → error (step 7), skip ticket.
   - Process each beneficiary INDEPENDENTLY, one at a time, through steps 3R–5 below. A failure on one person does NOT abort the others; record it and continue.
   - Also parse, per person: the tool(s), environment (Dev/Stage/Prod/Test if any), and the type (from the label).

3R. RESOLVE ONE PERSON (canonical key = accountId):
   a. Resolve via lookupJiraAccountId (by email if the ticket gives one, else full name).
   b. One match → use its accountId + exact displayName (as page title). For a REMOVAL, a deactivated/inactive single match is fine.
   c. Multiple matches → DISAMBIGUATE, don't give up:
      - Parse the person into first name + surname/initial.
      - Keep real active humans: accountType "atlassian", active==true, and a matching corporate email when visible; drop app/customer/deactivated accounts and bare-first-name-only accounts when a surname/initial was given.
      - Keep those whose first name matches AND surname matches (or begins with the given initial).
      - Confirm via an existing "Access Control" child page with a matching title.
      - One survivor → use it. Two+ equally-plausible real users = genuine collision → error (step 7), listing candidate names/accountIds/emails.
   d. DEPARTED-EMPLOYEE FALLBACK (REMOVAL only): if lookupJiraAccountId returns ZERO matches, don't error yet. Search with CQL `space = {confluence.spaceKey} AND parent = {confluence.parentPageId} AND title = "<name>"` (direct child of the Access Control parent only):
      - Exactly one page → adopt its exact title as canonical identity (no accountId — expected) and update its ACCESS REMOVED cell. Do NOT create a page.
      - Zero or multiple pages → error (step 7) for this person.
   e. Otherwise (zero matches on a GRANT, genuine collision, or person undeterminable) → error (step 7) for this person.

4. FIND THIS PERSON'S PAGE (match by displayName; Confluence page labels aren't used for matching):
   a. CQL `space = {confluence.spaceKey} AND parent = {confluence.parentPageId} AND title = "<displayName>"` — must be a direct child of the Access Control parent (pageId {confluence.parentPageId}, key {confluence.spaceKey}). Never match on title alone across the whole space.
   b. One match → use it.
   c. None → create it as a child of the parent page, titled with displayName, using FORMAT (name/role/dates header + standard table). EXCEPTION: never create in the departed-employee fallback (3R.d) — "none" there is an error (step 7).
   d. Multiple → error (step 7) for this person.

5. UPDATE THIS PERSON'S TABLE (regenerate whole; never surgically edit):
   a. Map each tool the ticket grants/removes for this person to exactly ONE row via TOOL MAPPING (one person may touch several rows). Reuse an existing row (incl. its env sub-rows) before adding one; add only per ADDING NEW ROWS. When in doubt, reuse.
   b. Read the existing table into a data model (Tool, Env, Details, Access Added, Access Removed). New page → start from the standard row set (see FORMAT). Preserve any existing custom rows.
   c. For each mapped row, update only the cell matching the ticket type (leave the other cell as-is); use statusCategory.key (not display name):
      - GRANT → ACCESS ADDED = ("YES" if key=="done" else "NO") + ` - <a href=TICKET_URL>KEY</a>`.
      - REMOVAL → ACCESS REMOVED, same YES/NO rule.
   d. Re-render the ENTIRE table from the model per FORMAT + TABLE STRUCTURE RULE, write it back whole.
   e. Re-fetch to confirm it saved and every row's cell count matches the structure rule; fix and re-verify if not.

6. UPDATE LABELS (writable) — only if statusCategory.key=="done" AND EVERY beneficiary on the ticket was processed successfully (page created/updated). If status isn't done, or ANY person errored, leave labels so a future run revisits it (pages for the people who succeeded stay updated).
   - GRANT done → remove `{labels.grant}`, add `{labels.grantDone}`.
   - REMOVAL done → remove `{labels.removal}` (and `{labels.grant}` if present), add `{labels.removalDone}`.
   Re-fetch labels to confirm the originating label is gone and the completion label present.

7. COMMENT ONLY ON ERRORS (per person: unresolvable beneficiary, genuine collision, multiple matching pages, a removal whose person is neither a resolvable user nor has a page, page write failed after retry, label update failed). Use ADF, @-mention the reviewer via a mention node with accountId `{reviewerAccountId}`, explain the issue and name WHICH beneficiary(ies) it concerns, list candidates for ambiguities. Combine all of a ticket's problems into ONE comment. A normal sync, a new row, or a processed removal is NOT an error. Before posting, read existing comments — don't duplicate a prior bot comment for the same issue.

8. Repeat for all tickets. Report: tickets skipped (already done / Backlog), and — counted by PERSON — grants processed, removals processed, beneficiaries flagged with errors, pages created vs updated, and any NEW ROWS added (person, tool, ticket key).

TOOL MAPPING (keyword in ticket → standard row; reuse existing rows first, add a new one only per ADDING NEW ROWS):
- Calendar / invite → Calendar Invites
- Jira → JIRA
- Confluence / wiki → Confluence
- Slack / channel → Slack Channels
- GitHub / repo → GitHub
- Platform / test / dev / stage / prod app access → Platform (Test/Dev/Stage/Prod)
- Figma → Figma
- Azure / Haptiq Platform (incl. any Azure sub-resource) → Azure - Haptiq Platform (Dev/Stage/Prod)
- Snowflake (incl. variants like "Snowflake Affinity") → Snowflake (Dev/Stage/Prod)
- Prefect → Prefect (Dev/Stage/Prod)
Map environment words (Dev/Stage/Prod/Test) to the matching env sub-row; if a tool ticket names no environment, use the tool's single/primary row.

ADDING NEW ROWS (only when the ticket fits no existing row and isn't a variant/feature/environment/rename of one — otherwise fold it in):
- Name it after the tool's canonical name from the ticket (Title Case, consistent with existing rows, e.g. "Databricks", "PagerDuty", "AWS - <account>"). Multiple envs → env sub-rows with a rowspanned Tool cell like Snowflake/Prefect; else a single row.
- Append after the standard rows, in first-encountered order. Once created, reuse it forever (never a second row for the same tool). Add it to another person's page only when their ticket references that tool.
- Keep the four-column structure and TABLE STRUCTURE RULE.

FORMAT (contentFormat html): page has a name/role/dates header, then ONE table: Tool(s) | Details | Access Added | Access Removed. Standard tool rows, always present in this order: Calendar Invites, JIRA, Confluence, Slack Channels, GitHub, Platform (Test/Dev/Stage/Prod), Figma, Azure - Haptiq Platform (Dev/Stage/Prod), Snowflake (Dev/Stage/Prod), Prefect (Dev/Stage/Prod). Any custom rows added via ADDING NEW ROWS appear after these. Parent "Access Control" page = pageId {confluence.parentPageId}; if `{confluence.referencePageUrl}` is set, use it as an existing person page to mirror the structure of.

TABLE STRUCTURE RULE (enforced by the full re-render in step 5d — verify every run): tools with multiple env rows use `rowspan` on the Tool cell. First row of a group = 4 `<td>`s (rowspan-tool, Details, Access Added, Access Removed). Every other row in that group = exactly 3 `<td>`s (Details, Access Added, Access Removed) — never 4. A single-row tool = 4 `<td>`s. Any description text goes together with the env name in one Details cell (e.g. "Dev - Snowflake Affinity DEV access"), never as its own extra cell.

Never change ticket status or any Jira fields besides the label updates in step 6 and the error comments in step 7.