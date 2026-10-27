# Smart Tag Skill

AI-assisted tagging of AEM assets, with author approval at every step and a full audit trail.

This repository is a Coworker marketplace containing the `smart-tag` plugin.

| Plugin | Skill | Description |
|--------|-------|-------------|
| `smart-tag` | `smart-tag-skill` | AI Smart Tagging for AEM assets with per-asset author approval, taxonomy add requests, and an Excel audit report. |

## What this skill does

Smart Tag Skill runs AI Smart Tagging over AEM Assets and helps an author apply the right controlled-vocabulary tags safely. It never creates taxonomy entries on its own and never applies a tag without explicit approval. Every suggestion is grounded in the asset's real content and matched against the live AEM tag taxonomy.

- Applies only approved + existing tags
- Routes missing tags to admins
- Excel audit report

## How it works

1. **Choose the scope.** Smart-tag an *entire folder*, a *single asset*, or a *filter condition* (e.g. all assets with `Asset Type = Logo` in a given folder).
2. **Resolve the asset set.** The skill lists the matching assets and confirms the count before doing anything.
3. **Process one asset at a time.** For each asset it reads current tags, runs AI Smart Tag logic, and shows current vs. suggested tags, then *waits for the author's approval before moving to the next asset*. If the physical image can't be fetched, it says so and gives the full asset URL including the AEM host, and suggestions are based on metadata only.
4. **Look-ahead (no waiting).** While the author is deciding on the current asset, the *next* asset's suggestions are already queried and prepared in the background. Metadata for the whole set is read and classified up front and cached, so each approval is instant. Look-ahead only pre-computes suggestions; it never pre-applies tags or batches approvals.
5. **Classify every suggestion.** Each suggested tag is marked **In taxonomy**, **Closest existing match**, or **Missing - new tag needed**. Existing tags are always preferred; a brand-new tag is a last resort.
6. **Act on the decision.** Approved tags that already exist in the taxonomy are applied to the asset (existing tags preserved). Approved tags that are missing are recorded as a taxonomy add-request, never applied directly.
7. **Produce the report.** At the end it exports an **Excel (.xlsx)** report of every asset processed, the tags applied, and the tags requested, plus a consolidated add-request for any missing tags.

## What the author gets

| Applied immediately | Routed for a new tag |
|---------------------|----------------------|
| Tags approved by the author that already exist in the master taxonomy are written straight to the asset, with each write verified against the API response. | Concepts with no close existing tag are collected into a taxonomy add-request so the master taxonomy stays clean and governed. New tags are expensive to add, so this is the exception. |

## Current taxonomy add-request path

```
Author approves missing tag
  -> Request document (Markdown + Excel)
  -> Email to taxonomy admins
  -> Admin adds tag manually
  -> Author applies tag
```

Today the hand-off to taxonomy administrators is a document the requestor forwards manually (email). It works, but it lives outside AEM and has no tracking.

## Next steps & recommendation

### Replace the email hand-off with an AEM Inbox workflow

When a new tag needs to be added to the taxonomy, invoke an **AEM workflow** that routes the request through the **AEM Inbox** to the **Taxonomy Administrators group**, instead of emailing named individuals. On approval, the workflow routes a task back to the **requestor's inbox**, who can then apply the newly created tag.

```
Requestor (submits via skill)
  -> AEM Workflow (taxonomy add-request)
  -> Admin group Inbox (Taxonomy Administrators)
  -> Approve & create tag
  -> Requestor Inbox (notified)
  -> Apply tag to asset
```

**Why this is better**

- **Group routing, not individuals:** requests go to a role-based Taxonomy Administrators group, so coverage survives staff changes and PTO.
- **Tracked & auditable:** every request, approval, and rejection is a work item in AEM with history, rather than an email thread.
- **Closed loop:** the requestor is notified in their own Inbox the moment the tag exists, and can apply it without chasing anyone.
- **Native to AEM:** no external system; uses the AEM Inbox authors already use for review/approval tasks.

**What it would take to implement**

| Piece | Detail |
|-------|--------|
| Workflow model | An AEM workflow model (e.g. *Taxonomy Tag Add Request*) with a Participant/Dialog Participant step assigned to the Taxonomy Administrators group, then a second step routed back to the initiator. |
| Admin group | An AEM group (e.g. `taxonomy-administrators`) containing the current and future taxonomy admins, with tag-create permission on `/content/cq:tags`. |
| Payload | The request carries the asset path, proposed tag title, target namespace, and rationale: the same fields the skill already captures. |
| Skill change | Swap the "email the admins" step for a call that starts the workflow instance; keep the Excel audit report. |
| Access needed | Author account needs permission to create/apply tags; the workflow author needs Workflow Console access. (The current account can apply existing tags but not create new ones, which is exactly why the governed workflow helps.) |

This is a recommendation, not something the skill does today. Standing up the AEM workflow model and admin group is an AEM administration task.

### Retrain the AI Tagging Model from author feedback

Give authors a way to feed corrections back to the model. During per-asset review the author can take **two** feedback actions, both captured as training signal:

- **Reject a suggested tag:** the AI suggestion was wrong or low quality for this asset.
- **Suggest a new tag:** a tag the AI missed that should apply to this asset.

Each event is appended to a **retrain list** persisted in the user's saved folder (`/.agent/saved/smart-tag-retrain-list.json`), keyed by asset path, so one asset can carry both a rejection and a suggestion. The list accumulates across sessions. Each entry records:

| Field | Example |
|-------|---------|
| assetPath | `/content/dam/.../<shoot-name>/image-001.jpg` |
| feedbackType | `rejected_tag` or `suggested_tag` |
| tag | the tag title rejected or suggested |
| reason / notes | why (optional, author free-text) |
| timestamp, user | when and by whom |

```
Author rejects tag (or suggests a new one)
  -> Append to retrain list (saved folder, JSON)
  -> List grows across sessions
  -> 10+ assets: assemble by tag
  -> Run AEM Smart Tag Training
  -> Check training report
```

**Does Adobe support retraining? Yes.** AEM Assets auto-applies Smart Tags using Adobe's AI image-recognition model. When it does not apply *your* business tags correctly, you train it on your own tag structure (**image assets only**; video and text use default Adobe AI and cannot be custom-trained). Once the retrain list covers more than **10 assets**, submit it like this:

1. **Assemble assets by tag.** Group the retrain-list assets under the tag they should train. A *suggested tag* becomes a tag to train; a *rejected tag* flags inaccurate tagging to correct.
2. **Create the training model.** In the AEM Assets Cloud Service UI go to **Assets > Smart Tag Training > Create**, add a Title and Description, and select the target tag(s) from `cq:tags`.
3. **Add example assets & submit.** Add the example images for each tag and Submit. Training status shows *Pending*, then *Trained* (or *Failed*).
4. **Verify.** Run the **Smart Tags Training report** (Tools > Assets > Reports) to see per-tag training status; it downloads as a spreadsheet.

**Rules & caveats (from Adobe documentation)**

- **Why the threshold is 10:** Adobe requires at least **10 images per tag** (recommends ~30, max 50) and at least 2 distinct tags per model, so a 10-asset trigger is the point there is enough signal to train.
- **Training cannot be reverted or removed:** once trained it sticks, so curate the example set carefully.
- **Moderation is not durable:** removing a wrong tag on a single asset is not remembered when the asset is reprocessed, which is exactly why retraining beats one-off tag removal.
- **Org scope:** training is auto-shared across environments in the same IMS org, but cannot be exported to a different org.
- **Image assets only:** the retrain list should collect image assets for training.

The reject / suggest capture and the persisted retrain list are a recommendation, not something the skill does today. Running AEM Smart Tag Training is an AEM Assets task performed by an author/admin with Smart Tag Training access.

## Demo use cases

### UC #1: Smart-tag all Photography assets in a folder

Filter-condition scope: every asset in a folder whose Asset Type is Photography.

```
run AI smart tag for all assets in
content/dam/<site>/<category>/<product>/photography/shoot/<shoot-name>
that are asset type = Photography
```

What happens:

- Resolves the set (e.g. 17 Photography assets) and confirms the count before processing.
- Processes each asset one at a time; the next asset's suggestions are prepared in the background so there is no wait between approvals.
- For each asset you pick which suggested tags to apply; existing taxonomy tags are applied, missing ones are recorded for the add-request.
- Ends with an Excel report of tags applied vs. requested per asset.

### UC #2: Smart-tag a single asset

Single-asset scope: one specific asset path.

```
AI smart tag
content/dam/<site>/<category>/<product>/photography/shoot/<shoot-name>/image-001.jpg
```

Suggests tags grounded in that asset's content, applies the approved existing tags, and produces a one-row report.

### UC #3: Smart-tag an entire folder (all asset types)

Folder scope: every asset under a folder, optionally including subfolders.

```
smart tag every asset in
content/dam/<site>/<category>/<product>/photography/shoot/<shoot-name>/
```

Same one-at-a-time flow with look-ahead, across the full folder.

## Invoking the skill

No command to memorize, just state the intent. The skill triggers on phrases like:

- "smart tag these assets" / "AI smart tag this folder"
- "AI tag the logos in this folder"
- "auto-tag this asset"
- "smart tag `<asset or folder>`"

## Install

### Option 1: Install from this marketplace (recommended)

1. In Coworker, go to Customizations > Marketplaces > Add Marketplace.
2. Enter this repository in `org/repo` format, branch `main`, and a unique name. Leave Subdirectory empty.
3. Once the status is **Ready**, go to Customizations > Plugins > Browse Marketplaces.
4. Select `smart-tag` and install.

Scope: whoever installs it. Installing never bypasses governance; Coworker still honors each user's Adobe product permissions, access controls, and approval gates.

### Option 2: Save it as a personal skill

For an individual user, with no admin needed.

1. Open `plugins/smart-tag/skills/smart-tag-skill/SKILL.md`.
2. In Coworker, paste its contents and say *"save this as a skill."*
3. It is then registered to that user's account and active in all their sessions.

Scope: personal, visible only to the account that saved it.

### Option 3: Org skill repository

For everyone in the organization. Requires an administrator: submit `SKILL.md` to the Coworker administrator / Adobe contact, who uploads it to the **organization skill repository** for review/approval. Once published, it appears for every user in the org.
