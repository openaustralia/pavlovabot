# PavlovaBot

A shared GitHub App and reusable workflow for openaustralia repos. Right now
it does one thing: on a Dependabot bundler PR, it regenerates Sorbet's
[Tapioca](https://github.com/Shopify/tapioca) RBI files and pushes the result
back to the PR, but only once it's confirmed the regenerated RBI still type
checks cleanly.

## Why this exists

Dependabot bumps a gem's version in `Gemfile.lock`, but it never runs
`tapioca gem` itself. The committed RBI then no longer matches the installed
gem, and `ci:type` fails with a "stale RBI" error that has nothing to do with
whether the gem bump actually broke anything. See
[openaustralia/planningalerts#2082](https://github.com/openaustralia/planningalerts/issues/2082)
and [planningalerts PR #2139](https://github.com/openaustralia/planningalerts/pull/2139)
for the investigation that led here.

This isn't the first attempt. An earlier version lived directly in
`planningalerts` ([PR #2043](https://github.com/openaustralia/planningalerts/pull/2043))
and was reverted two weeks later
([PR #2048](https://github.com/openaustralia/planningalerts/pull/2048))
because it pushed its fix-up commit using the default `GITHUB_TOKEN`, and
GitHub does not let a commit pushed with that token re-trigger other
`pull_request` workflows - so `test`, `type_check` and `lint` silently never
ran again against the final commit. PavlovaBot exists to push as a real,
separate identity instead, so those checks genuinely re-run.

## How it works

1. A Dependabot bundler PR triggers the adopting repo's caller workflow.
2. The job mints a short-lived PavlovaBot installation token
   (`actions/create-github-app-token`), regenerates the RBI
   (`bin/tapioca gem`, `bin/tapioca dsl`), and runs `bin/srb` against it.
3. **If `bin/srb` fails** - a real Sorbet error, not just a stale file - the
   job fails and pushes nothing. The branch is left exactly as Dependabot
   created it. A human needs to look at it, the same as any other failing
   check.
4. **If it's clean**, the job commits the regenerated RBI and pushes it using
   the PavlovaBot token. Because that's a real push (not `GITHUB_TOKEN`), it
   re-triggers the repo's normal CI, which becomes the actual gate on
   mergeability - PavlovaBot's own pre-check is just what decides whether to
   bother pushing.

### Why Dependabot secrets, not Actions secrets

GitHub treats any workflow run whose actor is `dependabot[bot]` as if it came
from a fork: the default `GITHUB_TOKEN` is read-only, and **ordinary
repository or organisation Actions secrets are not available to it at all**.
The separate
[Dependabot secrets](https://docs.github.com/en/code-security/reference/supply-chain-security/troubleshoot-dependabot/dependabot-on-actions)
store (Settings → Secrets and variables → **Dependabot**, not Actions) is
what's actually available to a Dependabot-triggered run, using the exact same
`${{ secrets.NAME }}` syntax. `PAVLOVABOT_APP_ID` and `PAVLOVABOT_PRIVATE_KEY`
have to live there, not in Actions secrets, or the workflow will silently
have no credentials the one time it actually runs.

The alternative GitHub offers - `pull_request_target`, which doesn't have
this restriction - was deliberately not used here. It runs with secrets
present while also checking out the PR's own content, and this job then runs
`bundle install`, which executes gem install scripts. That combination is the
"pwn request" shape GitHub's own security guidance warns against.

**Not yet independently verified**: whether `secrets: inherit` (or explicitly
forwarded secrets, as this workflow uses) correctly carries a caller's
Dependabot secrets through to a *reusable* called workflow, specifically in a
`dependabot[bot]`-actored run. This isn't a combination anyone seems to have
written up. Worth confirming on a real Dependabot PR before trusting it
fully, rather than assuming it, since the earlier revert was itself caused by
trusting an untested assumption about how Dependabot-triggered runs behave.

## Setting up the App (one-off, for an org admin)

GitHub requires a human to click through App creation even with a manifest,
so this can't be automated:

1. Go to `https://github.com/organizations/openaustralia/settings/apps/new`.
2. Use [`pavlovabot-app-manifest.json`](./pavlovabot-app-manifest.json) in
   this repo to fill in the form (or paste its values in by hand: repository
   permission **Contents: Read and write**, webhook **inactive**, "Where can
   this be installed" set to **Only on this account**, matching the org-only
   decision this App was set up under).
3. Generate a private key for the App from its settings page.
4. Install the App on whichever openaustralia repos need it (starting with
   `planningalerts`).
5. Add `PAVLOVABOT_APP_ID` and `PAVLOVABOT_PRIVATE_KEY` as **organisation**
   Dependabot secrets, scoped to selected repositories rather than all of
   them, and add repos to that list as they adopt this.

## Adopting this in a new repo

**Current access:** only `openaustralia/planningalerts` has been granted the
`PAVLOVABOT_APP_ID` and `PAVLOVABOT_PRIVATE_KEY` secrets, and only it has the
App installed. The workflow will not work in any other repo until these steps
are done, the first two by an org admin:

1. Install the App on the repo. Open the
   [PavlovaBot installation settings](https://github.com/organizations/openaustralia/settings/installations/163111443)
   and add the repo under "Repository access".
2. Grant the repo access to both secrets. Open the
   [organisation Dependabot secrets](https://github.com/organizations/openaustralia/settings/secrets/dependabot),
   edit `PAVLOVABOT_APP_ID` and `PAVLOVABOT_PRIVATE_KEY`, and add the repo to
   each one's "Selected repositories". Use the **Dependabot** tab, not Actions.
   Leave both secrets on "Selected repositories". Do not switch them to "All
   repositories", because the private key can push to every repo the App is
   installed on.
3. Add a caller workflow, e.g. `.github/workflows/dependabot-tapioca.yml`:

   ```yaml
   name: Dependabot Tapioca

   on:
     pull_request:
       types: [opened, synchronize, reopened]

   permissions:
     contents: read

   jobs:
     update_tapioca:
       if: >
         github.actor == 'dependabot[bot]' &&
         startsWith(github.head_ref, 'dependabot/bundler/') &&
         github.event.pull_request.head.repo.full_name == github.repository
       uses: openaustralia/pavlovabot/.github/workflows/dependabot-tapioca.yml@main
       secrets:
         app-id: ${{ secrets.PAVLOVABOT_APP_ID }}
         private-key: ${{ secrets.PAVLOVABOT_PRIVATE_KEY }}
   ```

4. Open a real Dependabot bundler PR (or wait for the next one) and check
   that PavlovaBot's job actually runs and, on success, that the repo's
   normal CI re-triggers on its push.
