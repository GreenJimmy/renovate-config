# GreenJimmy Renovate preset

One dependency policy for every GreenJimmy site. Change it here, every repo
picks it up — including sites that don't exist yet.

Each app's entire Renovate config is:

```json
{ "extends": ["github>GreenJimmy/renovate-config"] }
```

Saved as `renovate.json` in the app's root.

This mirrors [`Athlonaut/renovate-config`](https://github.com/Athlonaut/renovate-config)
deliberately, so the two fleets behave the same way. Where the policies differ,
it is written down below.

## What the policy does

| Behaviour | Why |
|---|---|
| Pull requests target **`dev`** | Matches the dev-to-main flow the sites are released through. |
| Non-major grouped into `production` / `development` | One pull request a week for routine bumps instead of a dozen. |
| Non-major **automerged** | CI is the gate. Majors always wait for a human. |
| Renovate merges its own pull requests (`platformAutomerge: false`) | GitHub's auto-merge cannot be enabled on a pull request that is already mergeable, and these branches carry no required status checks — so platform auto-merge silently never completes. Renovate merging through the API works regardless. |
| No `transitiveRemediation` | It took the Athlonaut fleet offline for a week. See below. |
| Majors need dashboard approval | The Next/Clerk/Prisma stack breaks on majors; each deserves reading. |
| `next` + `react` + types grouped | They move as a set; splitting them produces unbuildable intermediate states. |
| `@clerk/*` grouped | Clerk ships auth changes across several packages at once. |
| `prisma` + `@prisma/*` grouped | Client and CLI must match or `generate` breaks. |
| `minimumReleaseAge: 1 day` | Catches bad publishes before they reach you, and matches the `minimumReleaseAge` pnpm enforces in `pnpm-workspace.yaml`. Renovate proposing something younger would produce a pull request pnpm then refuses to install. |
| Security alerts bypass schedule and release-age | A fix you're waiting on shouldn't sit until Monday. |
| Weekly, Monday before 6am | Updates are waiting when the week starts, not landing mid-flow. |
| Dependency Dashboard | One issue listing everything pending, instead of triaging pull requests. |

## Three version ceilings

These are not preferences. Each one is a breakage that was hit:

| Pin | Reason | Lift when |
|---|---|---|
| `eslint`, `@eslint/js` `<10.0.0` | `eslint-config-next` pulls `eslint-plugin-react` 7.37.x, which peers on eslint <=9 and calls the `context.getFilename()` API ESLint 10 removed. | `eslint-config-next` ships an ESLint 10 compatible react plugin. |
| `pg` `<8.23.0` | pg 8.23.0 puts a non-string into the startup packet when driven by `@prisma/adapter-pg`, so every connection dies in `pg-protocol` `addCString` with `ERR_INVALID_ARG_TYPE`. | The two agree again. |
| `typescript` `<6.1.0` | `typescript-eslint` peers on `typescript <6.1.0`, so TS 7 breaks `pnpm lint`. | `typescript-eslint` ships TS 7 support (typescript-eslint#10940). |

Check whether each still applies before assuming it earns its place.

## Per-repo overrides

None today. Every repo has a `dev` branch and a CI workflow, so the extends line
is the whole config everywhere.

Two overrides used to be needed and are worth knowing about in case either
condition comes back:

- A repo with **no `dev` branch** must point Renovate at `main`, or it targets a
  branch that isn't there:

  ```json
  { "extends": ["github>GreenJimmy/renovate-config"], "baseBranchPatterns": ["main"] }
  ```

- A repo with **no CI workflow** must turn automerge off, or this preset merges
  dependency updates with nothing verifying them:

  ```json
  { "packageRules": [{ "matchPackageNames": ["*"], "automerge": false }] }
  ```

  `coyote-coaching` and `easy-ballot` carried this until 2026-09; both now have a
  CI workflow that runs on pull requests to `dev`, which is the gate automerge
  relies on.

Two settings that belong to the repo rather than to `renovate.json`: Issues must
be enabled (the Dependency Dashboard is an issue, and majors can only be approved
from it), and the Mend app must be installed on the repo.

## Don't add `transitiveRemediation`

Inherited warning from the Athlonaut preset, which is worth keeping in front of
whoever edits this next. It was added there on 2026-08-10 and every hosted run
after it died with `kernel-out-of-memory`; no repo got a Renovate run for the
following week, and six high-severity advisories sat untouched.

The lockfile-only gap it closes is real, and is covered instead by
version-scoped `overrides` in each app's `pnpm-workspace.yaml` — which is how the
`mysql2` and `deepmerge-ts` advisories are handled today.

## Running it

The **Mend-hosted Renovate app** is installed on the GreenJimmy account and runs
it. No token or scheduled workflow is needed — the app authenticates as itself.

**This repo is public** so the app (and Renovate's preset resolution) can read
`github>GreenJimmy/renovate-config` without extra access grants. It holds only
dependency policy — no secrets. Keep it that way.

## Migrating an app off Dependabot

1. Add `renovate.json` with the extends line above.
2. Let one Renovate cycle run and confirm the pull requests look right.
3. Only then delete `.github/dependabot.yml`.
4. **Also turn off Dependabot security updates** — a separate repo setting that
   `dependabot.yml` does not control:

```sh
gh api -X PUT repos/GreenJimmy/<repo>/vulnerability-alerts          # alerts: keep on
gh api -X DELETE repos/GreenJimmy/<repo>/automated-security-fixes   # the PR bot: off
```

Running both briefly is noisy but safe. Deleting Dependabot first leaves a gap
where nothing is watching for security updates.

Leave Dependabot **alerts** enabled. They are the detection feed Renovate's
`vulnerabilityAlerts` rule reads, so disabling them makes Renovate blind to
security issues rather than tidier.
