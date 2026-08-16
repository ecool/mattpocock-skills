# Issue tracker: Azure DevOps

Issues and specs for this repo live as Azure Boards work items. Use the Azure CLI (`az boards`, from the `azure-devops` extension) for all operations.

- **Organization**: `https://dev.azure.com/<org>`
- **Project**: `<project>`
- **Default work item type**: `Issue`
- **Closed state**: `Closed` _(Basic process uses `Done` — edit this line to match.)_

Triage "labels" are **tags** on the work item (`System.Tags`).

## Conventions

- **Create a work item**: `az boards work-item create --type Issue --title "..." --description "<p>...</p>"`. Descriptions are HTML, not markdown — wrap paragraphs in `<p>`, lists in `<ul><li>`. Use a heredoc for multi-line HTML.
- **Read a work item**: `az boards work-item show --id <id> --expand all`. Comments are a separate resource: `az devops invoke --area wit --resource comments --route-parameters project=<project> workItemId=<id> --api-version 7.1-preview.4`.
- **List work items**: `az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.Tags], [System.AssignedTo] FROM WorkItems WHERE [System.TeamProject] = @project AND [System.WorkItemType] = 'Issue' AND [System.State] <> 'Closed'"`. Filter tags with `[System.Tags] CONTAINS 'needs-triage'`.
- **Comment on a work item**: `az boards work-item update --id <id> --discussion "..."`.
- **Apply / remove tags**: read the current `System.Tags` first, then write the full new set — `--fields` **replaces** the tag list. `az boards work-item update --id <id> --fields "System.Tags=needs-triage; ready-for-agent"`.
- **Close**: `az boards work-item update --id <id> --state Closed --discussion "..."`.
- **Pull requests**: Azure DevOps calls them pull requests. Use `az repos pr create`, `az repos pr show`, `az repos pr list`, `az repos pr update`.

Infer org and project from `git remote -v` — `https://dev.azure.com/<org>/<project>/_git/<repo>`, `git@ssh.dev.azure.com:v3/<org>/<project>/<repo>`, or the `*.visualstudio.com` / `vs-ssh.visualstudio.com` equivalents. `az` detects these when run inside the clone. If it doesn't, set defaults once: `az devops configure --defaults organization=https://dev.azure.com/<org> project=<project>`.

Auth is `az login`, or a PAT in `AZURE_DEVOPS_EXT_PAT` / `az devops login`. The `azure-devops` extension installs itself on first `az boards` call.

Work item IDs are unique across the organization, not per project.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external pull requests as feature requests; `/triage` reads this flag.)_

When set to `yes`, PRs run through the same tags and states as work items, using `az repos pr`:

- **Read a PR**: `az repos pr show --id <id>`. Threads: `az devops invoke --area git --resource pullRequestThreads --route-parameters project=<project> repositoryId=<repo> pullRequestId=<id> --api-version 7.1`.
- **List external PRs for triage**: `az repos pr list --status active -o json`, then keep only PRs whose `createdBy` is not a project member/admin (a contributor's PR, not a team member's in-flight work).
- **Comment / tag / close**: post a thread via `az devops invoke` as above; PR labels are a separate REST surface from work-item tags (apply the same role strings there); abandon with `az repos pr update --id <id> --status abandoned`.

Unlike GitHub, Azure DevOps numbers work items and PRs separately, so `#42` is unambiguous once you know which surface the maintainer means.

## When a skill says "publish to the issue tracker"

Create an Azure Boards work item of the default type above.

## When a skill says "fetch the relevant ticket"

Run `az boards work-item show --id <id> --expand all` and fetch comments as above.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single work item with **child** work items as tickets.

- **Map**: a single work item tagged `wayfinder:map`, holding the Notes / Decisions-so-far / Fog body. `az boards work-item create --type Issue --title "..." --fields "System.Tags=wayfinder:map"`.
- **Child ticket**: a work item linked as a child of the map (`az boards work-item relation add --id <child> --relation-type parent --target-id <map>`) and tagged `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: Azure Boards' **native Predecessor/Successor** link — the canonical, UI-visible representation. A ticket blocked by `#n` gets `#n` as a predecessor: `az boards work-item relation add --id <child> --relation-type predecessor --target-id <blocker>`. Where a process template has disabled these links, fall back to a `Blocked by: #<n>, #<n>` line at the top of the HTML description. A ticket is unblocked when every blocker is closed.
- **Frontier query**: show the map with `--expand relations`, walk its children, drop any with an open predecessor or an assignee; first in map order wins.
- **Claim**: `az boards work-item update --id <n> --assigned-to "$(az ad signed-in-user show --query userPrincipalName -o tsv)"` — the session's first write. If `az ad` is unavailable, use the identity that owns the PAT.
- **Resolve**: `az boards work-item update --id <n> --discussion "<answer>" --state Closed`, then append a context pointer (gist + link) to the map's Decisions-so-far.
