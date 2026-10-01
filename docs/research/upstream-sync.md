# Reviewed upstream sync at 09:00 Europe/London

Research for [Design reviewed upstream sync at 09:00 Europe/London](https://github.com/Qiuyi-Hong/addyosmani-skills/issues/4), checked 1 October 2026. These are facts and a recommended design, not a deployed workflow. Ownership, import scope, and the reliability contract remain human decisions in [Choose upstream ownership and the sync reliability contract](https://github.com/Qiuyi-Hong/addyosmani-skills/issues/5).

## Schedule: native time zones now work

GitHub's current workflow syntax supports an IANA `timezone` alongside each cron entry. The intended trigger can therefore be expressed directly:

```yaml
on:
  schedule:
    - cron: '0 9 * * *'
      timezone: Europe/London
  workflow_dispatch:
```

This schedules 09:00 London local time throughout GMT and BST; no dual UTC schedule or seasonal date guard is needed. The workflow must exist on the default branch and scheduled runs use that branch's latest commit. [GitHub workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onschedule), [schedule event](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule).

The trigger is best effort. GitHub documents delays and possible dropped jobs during high load, particularly at the start of an hour. Keep 09:00 because the user specified it, but do not describe it as an exact start-time guarantee. [GitHub workflow troubleshooting](https://docs.github.com/en/actions/how-tos/troubleshoot-workflows#scheduled-workflows).

Public-repository schedules are automatically disabled after 60 days without repository activity. Workflow execution alone is not documented as preventing that condition. Re-enabling is possible through the UI, REST API, or `gh workflow enable`; an inactive schedule cannot run its own recovery. [Disabling and enabling workflows](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/disable-and-enable-workflows).

**Recommended default:** native schedule plus manual dispatch for recovery. This only satisfies a best-effort daily checking contract. Unattended detection after long inactivity needs an independently running watchdog/dispatcher, which is a separate human choice. Do not add empty commits or an external scheduler silently. Manual merging also means `main` is current only through the latest accepted update, rather than always identical to upstream.

## Observed repository prerequisites

Read-only API observations on 1 October 2026:

| Observation | Value | Primary source |
| --- | --- | --- |
| Downstream repository | Public, independent repository; `fork: false`; default branch `main` | [Repository API](https://api.github.com/repos/Qiuyi-Hong/addyosmani-skills) |
| Downstream main | `c8dd0af6c7b8b1d4fd792b28c304bf5bf62b1462`; unprotected | [Branch API](https://api.github.com/repos/Qiuyi-Hong/addyosmani-skills/branches/main) |
| Authenticated maintainer | `Qiuyi-Hong`, repository admin access | [Repository API](https://api.github.com/repos/Qiuyi-Hong/addyosmani-skills) |
| Actions policy | Enabled; all actions allowed; SHA pinning not enforced | [Actions permissions API](https://api.github.com/repos/Qiuyi-Hong/addyosmani-skills/actions/permissions) |
| Default workflow token policy | `default_workflow_permissions: read`; `can_approve_pull_request_reviews: false` | [Workflow permissions API](https://api.github.com/repos/Qiuyi-Hong/addyosmani-skills/actions/permissions/workflow) |
| Existing workflows | None | [Workflows API](https://api.github.com/repos/Qiuyi-Hong/addyosmani-skills/actions/workflows) |
| Automatic merge | Disabled | [Repository API](https://api.github.com/repos/Qiuyi-Hong/addyosmani-skills) |
| Upstream | Public; default branch `main`; head `2686b620fc1fed2e8f60c704839c766b8594c6b6` | [Repository API](https://api.github.com/repos/addyosmani/agent-skills), [pinned commit](https://github.com/addyosmani/agent-skills/commit/2686b620fc1fed2e8f60c704839c766b8594c6b6) |

The documented GitHub fork-sync operation targets a fork and updates its branch; it does not provide this independent repository's adaptation and reviewed-PR contract. A bot branch inside the downstream repository is sufficient, without changing GitHub fork relationships or merging unrelated histories. [Syncing a fork](https://docs.github.com/en/pull-requests/how-tos/work-with-forks/syncing-a-fork), [PR head/base requirements](https://docs.github.com/en/rest/pulls/pulls#create-a-pull-request).

No settings or production files were changed. Notification preferences could not be inspected with the existing token: the subscription API requires the `notifications` scope. Do not infer enabled delivery from repository ownership.

## Recommended import model, conditional on generated-file ownership

Fetch pristine upstream into a temporary directory, resolve one source commit before reading any files, and build the downstream package using trusted adaptation code from downstream `main`. Record source repository, branch, exact commit SHA, and the paths owned by generation in one provenance file. The copy on `main` is the **accepted** pin; a PR contains a **proposed** pin until manually merged.

Do not keep a second live `SKILL.md` tree in the distributable checkout. Skills CLI 1.7.0 normally searches conventional containers, but `--full-depth` or an empty conventional search invokes recursive discovery. Dot-prefixed directories are not excluded, and names normally deduplicate by first discovery. Temporary source avoids ambiguity without an archive or another Git ref. [Pinned discovery implementation](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/skills.ts#L254), [recursive walker and excluded directories](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/skills.ts#L10).

Upstream already contains 25 base skills, nine `.claude/commands` entries, Codex plugin metadata, and shared references. Reuse that material and adapt only the incompatibilities established by the command and installation research. Import scope must be explicit: copying the entire upstream root would also replace downstream `AGENTS.md`, README, tracker guidance, and workflows. [Pinned upstream tree](https://github.com/addyosmani/agent-skills/tree/2686b620fc1fed2e8f60c704839c766b8594c6b6).

Suggested ownership boundary:

- **Generated:** imported/adapted skill files, converted command-entry skills, necessary skill-local resources and license notices, and provenance/owned-path metadata.
- **Local:** adaptation code, explicit overrides, installation identities and marketplace configuration, README, `AGENTS.md`, tracker/domain/research docs, and `.github/workflows`.
- **Excluded by default:** unrelated agent integrations, upstream repository automation, and documentation not needed by the installed collection. A full mirror is a human scope choice, not a prerequisite for installing these skills.

Skills CLI copies the selected skill directory rather than the repository root. Required references and notices must therefore travel within each selected skill. Retain upstream's MIT copyright and permission notice. [Pinned installer](https://github.com/vercel-labs/skills/blob/3694740352eeef5cdd689af694c485f1ff62eec3/src/installer.ts#L352), [upstream license](https://github.com/addyosmani/agent-skills/blob/2686b620fc1fed2e8f60c704839c766b8594c6b6/LICENSE).

Generation compares the old owned-path set with the new one: additions create files, modifications replace owned files, renames remove the old generated path and create the new one, and deletions remove only previously owned paths. Never recursively delete an entire shared directory. Name collisions, unsupported source formats, missing required resources, or changes to a protected/local path stop the proposal. Local edits survive by living in the adapter or explicit override inputs; arbitrary direct edits to generated files require a different preservation model. Detect unexpected manual edits to owned files or the bot branch instead of silently discarding them.

An upstream force-push or default-branch change must be detected and reported. If the accepted source cannot be fetched or its lineage verified, retain the accepted package/pin and require maintainer review before rebaselining. This is a deliberate safe-failure boundary: source is auditable through pinned upstream links while reachable, not guaranteed available offline after upstream history removal. Add retained archives or a source ref only if the maintainer requires that stronger guarantee. Git can compare two trees directly; rename reporting is useful presentation, not necessary for correct replacement/deletion ownership. [Git diff](https://git-scm.com/docs/git-diff).

## Recommended PR lifecycle

Use Git and the existing GitHub CLI rather than a custom PR client. Keep one bot-owned branch, for example `codex/sync-upstream`, and at most one open rolling PR into downstream `main`. Serialize mutation with a fixed concurrency group and `cancel-in-progress: false`; GitHub permits one running member of the group. Query PRs by explicit repository, base, head, and state, rather than title. [GitHub concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency), [PR filtering API](https://docs.github.com/en/rest/pulls/pulls#list-pull-requests).

| State | Recommended behavior |
| --- | --- |
| Upstream SHA already accepted; package unchanged | Successful quiet no-op: no push, PR, or comment. |
| New source SHA, no open sync PR | Generate and validate; push only the sync branch; create one PR. |
| Open sync PR, same proposed SHA/output | Quiet no-op; recover any incomplete metadata/notification operation idempotently. |
| Open sync PR, newer source SHA or changed downstream adaptation | Rebuild against current `main`; update the same PR; invalidate old validation; notify only for a material update. |
| Sync PR manually merged | The provenance pin on `main` becomes accepted; next run uses it. |
| Sync PR closed without merge | Treat the recorded proposed SHA as rejected. Do not reopen/recreate that proposal each day. New upstream SHA may produce a new cumulative PR; state explicitly if it includes previously rejected changes. |
| Maintainer wants to reconsider the same rejected SHA | Explicitly reopen or use an intentional retry control; an ordinary rerun is not consent to retry a rejection. |
| Existing PR becomes unnecessary | Close it only after confirming its generated diff is empty against current `main`. |
| Source/network/validation/permission failure | Fail visibly; preserve `main`, accepted pin, and the last valid proposal; do not publish a partial package as ready. |

Keep the proposed source SHA in managed PR metadata and the branch commit so rejected-source suppression survives branch deletion and squash merging. Before replacing the bot branch, verify its expected tip and use a lease; unexpected human commits require intervention. Never force-push `main`. A create/push failure followed by a rerun must reuse the existing branch/PR, not produce duplicates. A rolling PR changing underneath review must identify the new source SHA and require validation of the new head.

PR content should include accepted/proposed source SHAs, immutable upstream commit and comparison links when available, source and adapted-file change summaries, validation results, and any previously rejected included changes. No merge API, automatic approval, or auto-merge step belongs in the sync workflow. Current absence of branch protection means manual merging is a workflow policy rather than an enforced independent-review requirement.

## Authentication, CI, and notification

**Minimum default:** use the repository-scoped, short-lived `GITHUB_TOKEN`. The sync job explicitly needs `contents: write` for its branch and `pull-requests: write` for PR operations; leave other scopes absent. Defaults can remain read-only. Future setup must enable **Allow GitHub Actions to create and approve pull requests** under Settings → Actions → General: the observed setting is currently false and prevents creation with this token. Enabling that coarse setting does not authorize adding automated approval code. [Token authentication](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token), [repository setting](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests).

Current GitHub documentation changed an old assumption: PRs created/updated with `GITHUB_TOKEN` **do** produce `pull_request` workflow runs for `opened`, `synchronize`, and `reopened`, but those runs require a writer to select **Approve workflows to run**. `push` workflows remain suppressed. Thus run trusted static validation before proposing, and use read-only `pull_request` validation of the proposed head with explicit maintainer approval. Never treat queued/unapproved checks as passed. [Current GITHUB_TOKEN event behavior](https://docs.github.com/en/actions/concepts/security/github_token#when-github_token-triggers-workflow-runs).

If automatically running PR CI is required, a repository-limited GitHub App installation token is the extension; current GitHub docs also permit a PAT. An App adds provisioning and a private-key secret. A PAT owned by the reviewer also makes that person the PR author, which may not suit formal approval. Do not introduce either merely to avoid one human CI approval. [GitHub workflow-trigger guidance](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow). Importing `.github/workflows` would also bring additional workflow-write credential requirements and unreviewed automation, so exclude it from generated sync outputs. [Repository contents permissions](https://docs.github.com/en/rest/repos/contents#create-or-update-file-contents).

Request review from `Qiuyi-Hong` when the bot PR is first created: review requests generate notifications. For a materially updated open PR, post one source-SHA-keyed summary comment mentioning `@Qiuyi-Hong`; check existing markers on reruns so the same update does not notify twice. No-change days stay quiet. GitHub delivery follows personal notification preferences; it cannot be guaranteed by a workflow. [Review-request notifications](https://docs.github.com/en/rest/pulls/review-requests#about-review-requests), [notification configuration](https://docs.github.com/en/subscriptions-and-notifications/get-started/configuring-notifications). CLI supports explicit head/base/repository, body files, and reviewers. [GitHub CLI PR creation](https://cli.github.com/manual/gh_pr_create).

Failures should produce an unsuccessful Actions run with useful recovery instructions. Setup must confirm the maintainer receives Actions failure notifications; scheduled notifications are associated with the last cron editor, not simply the repository owner. A named, deduplicated failure issue is an optional stronger alert if those preferences cannot satisfy the agreed contract, and requires `issues: write`. Silent schedule disablement requires the independent monitoring choice above. [Schedule actor/notifications](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#actor-for-scheduled-workflows), [Actions notification preferences](https://docs.github.com/en/subscriptions-and-notifications/how-tos/managing-github-actions-notifications).

Treat upstream as data in the privileged sync job. Do not execute newly fetched upstream scripts, hooks, or workflows with write credentials. Use trusted downstream validators; reject paths escaping permitted roots and link-based escapes. Pin any Actions dependencies to reviewed full commit SHAs. [GitHub secure-use guidance](https://docs.github.com/en/actions/reference/security/secure-use).

## Minimum acceptance evidence for later implementation

This runnable standard-library check passed during research; it verifies the local-time/UTC expectation around both 2026 UK transitions, not GitHub dispatch punctuality:

```sh
python3 - <<'PY'
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

for day, utc_hour in [('2026-03-28', 9), ('2026-03-29', 8),
                      ('2026-10-24', 8), ('2026-10-25', 9)]:
    local = datetime.fromisoformat(day + 'T09:00:00').replace(tzinfo=ZoneInfo('Europe/London'))
    utc = local.astimezone(timezone.utc)
    assert utc.hour == utc_hour and utc.minute == 0, (local, utc)
print('GMT/BST schedule expectation passed')
PY
```

Later implementation should leave one small runnable fixture check for the importer and PR-state decisions, rather than a test framework for each operation:

1. Simulate source add/modify/rename/delete; assert exact generated paths/content, removed stale entries, retained local guidance/overrides, self-contained resources, and license notices.
2. Run twice on the same source/base; assert identical files and no second push/PR/comment. Simulate open, merged, and rejected proposals and verify the lifecycle table, including rejection after branch deletion.
3. Inject name/path collisions, link escapes, inaccessible old source/history rewrite, API denial, invalid artifacts, and unexpected human branch commits; assert failure leaves the accepted package/pin intact.
4. In a disposable repository at implementation time, create/update a bot PR with the chosen credential; verify actual CI approval behavior, notification, rerun recovery, serialized concurrent dispatch, and that only manual merging changes `main`.

No live sync, notification, credential, or production CI test was performed in this decision-only research session.
