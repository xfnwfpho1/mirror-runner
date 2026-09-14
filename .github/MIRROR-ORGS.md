# MIRROR-ORGS runbook (G-14 — org-repo mirror coverage)

The catch-all mirror originally covered PERSONAL repos only
(`user/repos?type=owner`). `MIRROR_ORGS` (repo var, **default empty = the old
personal-only behavior, zero risk**) extends the plan job to org repos:

```
MIRROR_ORGS = "claudecode-headless,extreme-velocity"     # comma-separated org logins
```

Org repos are enumerated via `/orgs/{org}/repos?type=all` (private included —
the PAT's scopes cover them), tagged with their scope, and mirrored to GitLab
**subgroups** `<ns>/gh-orgs/<org>/<repo>` on every configured GitLab target —
collision-free by construction (the flat namespace already has an
org-vs-personal `cc-gha-exploration` name clash; subgroups fix the class of
problem). The `fsm-state` branch rides along automatically (refmap = all
heads+tags), so the FSM journal/state history gains an off-platform copy.

## Subgroups are NEVER auto-created (by design)

R3 review (BLOCKING): nothing in the kit creates GitLab GROUPS — only
projects (`POST /projects`). A project create under a nonexistent namespace
fails. Rather than auto-creating groups from a CI runner (an unreviewed
structural mutation of the backup target), the workflow **skips** an org's
repos while the subgroup is missing:

- the run logs `::warning::org mirror: org '<org>' SKIPPED — GitLab subgroup
  'gh-orgs/<org>' missing` (public log: org name only — the target namespace
  stays masked),
- one deduped note per 24h lands on the private meta repo's open
  `catchall-failure` tracking issue with the FULL subgroup paths,
- the run itself stays GREEN (a missing subgroup is an operator action item,
  not a mirror failure) and the repos pick up automatically on the next cycle
  once the subgroup exists.

## Operator action: create the subgroups (one-time, per org × target)

GitLab UI: open the target group → **New group → subgroup** →
name/path `gh-orgs` (once per namespace), then `gh-orgs/<org>` under it.

Or API (from any machine holding a group-owner PAT):

```bash
# 1. find the parent group id
curl -s -H "PRIVATE-TOKEN: $GL_PAT" "https://gitlab.com/api/v4/groups/<ns-url-encoded>" | jq .id
# 2. create gh-orgs (skip if it already exists)
curl -X POST -H "PRIVATE-TOKEN: $GL_PAT" "https://gitlab.com/api/v4/groups" \
  -d "name=gh-orgs" -d "path=gh-orgs" -d "parent_id=<ns-id>"
# 3. create gh-orgs/<org> under it
PARENT=$(curl -s -H "PRIVATE-TOKEN: $GL_PAT" "https://gitlab.com/api/v4/groups/<ns>%2Fgh-orgs" | jq .id)
curl -X POST -H "PRIVATE-TOKEN: $GL_PAT" "https://gitlab.com/api/v4/groups" \
  -d "name=<org>" -d "path=<org>" -d "parent_id=${PARENT}"
```

Repeat per GitLab target (e.g. both the central and the per-account lane).
The mirror WAF lottery (~1/3 of HK-IP requests 403) applies — retry on 403.

## After subgroup creation

1. Next hourly run: the plan job sees the subgroups, includes the org repos
   (they will read as `drift` — new projects), and the mirror job creates each
   project via the kit's existing `POST /projects` (`namespace_id` = the full
   subgroup path) and pushes. If a project auto-create is rejected, the job
   fails LOUDLY (the kit contract) — create the project manually under the
   subgroup with the same name and re-run.
2. Verify with one restore drill on an org repo (the kit's daily verify drill
   covers matrix jobs automatically; force it with a manual
   `workflow_dispatch` run if wanted).
3. Retire the old MANUAL flat org mirrors (`privatimail-llc-group/fsm-lab`,
   `cc-gha-exploration`) by GitLab rename/archive AFTER one green verify cycle
   on the automated `gh-orgs/*` chain (Q8).

## Notes / limits (v1)

- Org mirroring requires every configured target to be `gitlab`-kind; with any
  non-gitlab target configured, org repos are skipped with a warning (never a
  failed run).
- `MIRROR_EXCLUDE` gains an `org:<login>` form (whole-org exclusion, useful to
  stage rollouts one org at a time) alongside the legacy bare-name and
  `owner/name` slug forms.
- Workload grows 15 → ~25 repos: plan timeout is 15 min; consider raising the
  mirror matrix `max-parallel` 2 → 3 at enablement time (one line, kept at 2
  pre-enablement for zero observable change).
- The meta status files gain `scopes[]` / per-repo detail in a follow-up
  (kit-side change — deferred; the counts-only shape is unchanged this wave).
