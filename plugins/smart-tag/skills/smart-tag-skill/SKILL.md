---
name: smart-tag-skill
description: Run AI Smart Tagging on AEM assets, show current vs. suggested tags, and process assets one at a time with author approval — applying tags already in the taxonomy or drafting a master-taxonomy add request for missing ones, then exporting an Excel report. Use when the user wants to smart tag, smart-tag, AI smart tag, AI tag, auto-tag, or smart tag AEM assets, or get AI tag suggestions for assets.
metadata:
  author: user
---

# Smart Tag Skill

Run AI Smart Tag logic on AEM assets. The author chooses the scope (a whole
folder, a single asset, or a filtered set). Assets are then processed **one at
a time**: for each asset, show current vs. suggested tags, get the author's
approval, then act — and **wait for approval before moving to the next asset**.
Act on each approved tag:
- Approved tag **already in the master taxonomy** → apply it to the asset.
- Approved tag **missing from the taxonomy** → do NOT apply; record it as a
  master-taxonomy add request for the taxonomy admins.

At the end, produce an **Excel (.xlsx) report** of every asset processed and,
for each, which tags were applied and which were requested.

## Configuration

**Taxonomy admins** (contact to add a missing tag to the master taxonomy):
- <Taxonomy admin 1 name> — <admin 1 email>
- <Taxonomy admin 2 name> — <admin 2 email>

## Workflow

### Step 1 — Gather inputs
Use `ask_user_question` to collect, unless already provided:
- **Environment** — the AEM author instance (host/URL or environment name).
- **Scope** — ask which of the following the author wants to smart-tag:
  1. **Entire folder** — every asset in a given `/content/dam` folder (ask
     whether to include subfolders).
  2. **A specific asset** — one asset path.
  3. **A filter condition** — assets in a folder matching a metadata condition,
     e.g. Asset Type = Logo, a given usage right, format, or any custom
     `<namespace>:*` / `dc:*` field value. Collect the folder, the field, and the value.

### Step 2 — Resolve the asset set
Using the AEM read API (service="aem", or `AEM__read-api`), build the list of
target assets for the chosen scope:
- **Folder / filter**: query `search-assets` filtered on
  `repositoryMetadata.repo:ancestors` for the folder, then apply the filter
  condition client-side (or as a term filter) to narrow the set.
- **Single asset**: filter on `repositoryMetadata.repo:path`.

Show the author the resolved list and **count** of assets, and confirm before
processing. If a path can't be resolved or the set is empty, say so plainly.

### Step 3 — Process assets ONE AT A TIME
Iterate the resolved assets in order. For **each** asset, do steps 3a–3d fully,
then STOP and wait for the author before starting the next asset. Never batch
approvals across assets.

**3a — Read the asset and current tags.** Capture name, path, format,
`dc:title`, `dc:description`, `dc:subject`, applied tags (`xcm:keywords`), and
existing Smart Tags (`xcm:machineKeywords`).

**3b — Run AI Smart Tag logic.** Derive suggested tags from the asset's actual
content (machine keywords, title, description, subject). Match each against the
live taxonomy (`/content/cq:tags/<namespace>.N.json`) and classify:
- **In taxonomy** — exact tag exists, ready to apply.
- **Closest existing match** — no exact match, but a reasonably close existing
  tag covers the concept (search synonyms, broader/parent, related terms before
  giving up). Recommend the existing tag; note the original concept.
- **Missing — new tag needed** — only when no reasonably close tag exists.

Prefer an existing tag over proposing a new one; adding a master-taxonomy tag is
complex and expensive, so a new tag is a last resort. Ground every suggestion in
real asset content and real taxonomy entries; never invent tags.

**3c — Present and get approval for THIS asset.** Show, inline, the asset's
current tags and the suggested tags with their classification (titles only,
never normalized tagNames). Then use `ask_user_question` (multi-select) for the
author to approve which suggested tags to proceed with for this asset. Allow
free-text additions.

**3d — Act on the approved tags for THIS asset.**
- **Approved AND in taxonomy** → apply it. Via the AEM write API
  (service="aem", or `AEM__write-api`), add the tag to the asset's `cq:tags` /
  `xcm:keywords`, preserving existing tags (add, don't overwrite). Verify
  success from the API response; never claim success on a failed call.
- **Approved BUT missing** → do NOT apply. Record it as a taxonomy add request
  for the admins.

**3e — Log the result** for this asset in a running report (see Step 4): asset
name, path, tags applied (titles), tags requested (titles), and status.

Then move to the next asset and repeat 3a–3e. After the last asset, go to
Step 4.

**Look-ahead (pipelining).** To avoid making the author wait, prepare the NEXT
asset's suggestions in the background while the author is deciding on the
current one. Concretely: read and analyze all assets' metadata up front (one
batched read), run the Smart Tag classification for every asset before the
approval loop starts, and cache the per-asset suggestion tables. Then the
approval loop only presents a pre-computed table and applies the result — no
read/analysis latency between assets. Always keep the suggestions for asset
N+1 ready by the time the author finishes asset N. Still apply tags and record
results strictly per-asset, one approval at a time; look-ahead only pre-computes
suggestions, it never pre-applies or batches approvals.

### Step 4 — Running report + Excel export
Maintain a running record across all processed assets with these columns:
`Asset Name`, `Asset Path`, `Current Tags`, `Tags Applied`, `Tags Requested
(missing)`, `Status`, `Timestamp`.

At the end, write the record to an **Excel .xlsx** file in the workspace using
`execute_code` (pandas + openpyxl, or openpyxl directly) and give the author a
download link via `get_download_url`. Also render a concise inline summary
(total assets, total tags applied, total tags requested). If any missing tags
were requested, also produce the master-taxonomy add request (template below),
consolidating all requested tags across assets, and provide it as a Markdown
file with a download link.

## Master-taxonomy add request template

```
Subject: Request to add new tag(s) to the master taxonomy

To: <Taxonomy admin 1 name> <<admin 1 email>>;
    <Taxonomy admin 2 name> <<admin 2 email>>

Hi <admin names>,

Please add the following tag(s) to the master taxonomy so they can be applied
to the assets below.

Requester: <current user name> <<current user email>>
Date: <today>
Environment: <AEM author instance>

Suggested tag(s) to add:
| Suggested tag | Proposed namespace | Asset(s) | Rationale |
|---------------|--------------------|----------|-----------|
| <tag title>   | <namespace>        | <asset>  | <why it fits> |

Once added, I will apply the tag(s) to the assets. Thank you.

<current user name>
```

## Rules
- Process assets strictly one at a time; wait for the author's approval on the
  current asset before touching the next. Never auto-apply across a batch.
- Apply tags ONLY when approved by the author AND already in the taxonomy. Never
  create taxonomy entries yourself.
- Always try to cover a concept with an existing tag first; route to the
  taxonomy admins only when no reasonably close existing tag can be found.
- Missing tags are never applied — they are recorded as add requests.
- Always show tag titles, never normalized tagNames.
- Confirm the environment and scope before any write.
- Report failures honestly; verify each apply from the API response.
- Always produce the final Excel report of applied vs. requested tags, even if
  only one asset was processed.
