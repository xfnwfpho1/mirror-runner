# mirror-runner

Public GitHub Actions runner for the account-wide repo backup catch-all.
**Public repos get free/unlimited Actions minutes**, so the schedule lives
here while everything sensitive stays private:

- core scripts: private `catchall-mirror-kit` repo (checked out at runtime via PAT)
- run details: private meta repo (`status/latest.json`)
- secrets: this repo's Actions secret store only (encrypted, masked in logs)

The workflow mirrors every repo in the account to GitLab + Netlify-blob
bundle chains every 6 hours. It prints only indexes/counts publicly.

See the private `catchall-mirror-kit` repo for full documentation.

> Note: `.last-run` is updated by every run — it keeps this repo "active" so
> GitHub does not auto-disable scheduled workflows after 60 days of inactivity.
