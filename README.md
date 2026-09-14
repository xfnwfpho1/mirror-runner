# mirror-runner

Public GitHub Actions runner for the account-wide repo backup catch-all.
**Public repos get free/unlimited Actions minutes**, so the schedule lives
here while everything sensitive stays private:

- core scripts: private `catchall-mirror-kit` repo (checked out at runtime via PAT)
- run details: private meta repo (`status/latest.json`)
- secrets: this repo's Actions secret store only (encrypted, masked in logs)

The workflow mirrors every repo in the account to GitLab + Netlify-blob
bundle chains hourly. It prints only indexes/counts publicly.

Org-repo coverage (G-14): set the `MIRROR_ORGS` repo var (comma-separated
org logins; **default empty = personal repos only — today's behavior**) to
also mirror org repos into GitLab subgroups `gh-orgs/<org>/<repo>` on every
configured GitLab target. Subgroups are NEVER auto-created — while one is
missing the run stays green, logs a warning, and posts one deduped note per
24h on the private meta tracking issue; the operator creates the subgroups
(runbook: `.github/MIRROR-ORGS.md`) and the repos mirror automatically on
the next cycle. `MIRROR_EXCLUDE` additionally accepts the `org:<login>`
whole-org form for staged rollouts.

See the private `catchall-mirror-kit` repo for full documentation.

> Note: `.last-run` is updated by every run — it keeps this repo "active" so
> GitHub does not auto-disable scheduled workflows after 60 days of inactivity.
