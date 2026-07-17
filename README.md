# Monad GitOps demo

Declarative, Git-driven management of a [Monad](https://app.monad.com) security
data pipeline, reconciled by an LLM. You edit JSON in this repo and open a PR; a
workflow posts a **plan**; on merge, a workflow **applies** the changes to the
live Monad org and commits an updated lockfile.

```
edit monad/*.json ──▶ open PR ──▶ [monad-plan] posts plan comment ──▶ review + merge ──▶ [monad-apply] changes Monad + updates lockfile
```

The reconciler is an LLM (Claude) following a fixed contract in
[`RECONCILE.md`](./RECONCILE.md). That file is the spec — read it to understand
exactly what the automation is allowed to do.

## Layout

```
monad/                     # DESIRED STATE — the only thing the reconciler reads
  inputs/                  #   one *.json per input connector
  transforms/              #   one *.json per transform
  outputs/                 #   one *.json per output/destination
  pipelines/               #   one *.json per pipeline (nodes + edges)
.monad-lock.json           # reconciler-managed: <kind>:<ref> -> Monad server id
RECONCILE.md               # the binding reconcile contract for the LLM
examples/                  # reference snippets — IGNORED by the reconciler
.github/workflows/
  plan.yml                 # PR: read-only plan, posted as a comment
  apply.yml                # merge to main: apply, then commit the lockfile
```

## The demo pipeline

`Cloudtrail`: AWS CloudTrail S3 input → **Drop Low-Value Fields** → **Drop
CloudTrail Duplicated Data** → **dev-null** sink.

The terminal sink is `dev-null` **on purpose** — this is a demonstration, so it
discards records rather than shipping them anywhere. `examples/elasticsearch-output.json`
shows how you'd swap in a real destination (with a secret) when you want one.

## Design decisions

- **Object identity via lockfile.** Monad component ids are server-generated, so
  the repo can't own them. `.monad-lock.json` maps each object's stable
  `<kind>:<ref>` to its Monad id. The reconciler updates by id and **never**
  adopts an object by name — that's what prevents duplicate components on
  re-runs. The lockfile here is seeded from the live `jfrog-417c` org, so a
  first apply against that org updates in place instead of creating copies.
- **`prune: true`.** Delete a JSON file and the reconciler deletes the
  corresponding Monad object (pipelines pruned before components).
- **Plan on PR, apply on merge.** Review sees exactly what will change before it
  changes. The apply only runs post-merge on `main`.
- **Sensitive values via GitHub Actions secrets, never in git.** A value in a
  component file — a true secret in `secrets`, or a sensitive identifier in
  `settings` (this repo is public, so the CloudTrail S3 bucket and IAM role ARN
  are handled this way) — is written as `"env:VAR_NAME"`. The apply workflow
  exports the matching GitHub secret to that env var; the reconciler resolves it
  at apply time and sends it inline to Monad. Values are never committed,
  printed, or logged. See `RECONCILE.md` §5.

## Repo setup (do this after `git init` + first push)

1. **Secrets** (Settings → Secrets and variables → Actions):
   - `ANTHROPIC_API_KEY` — for the LLM reconciler.
   - `MONAD_API_KEY` — a Monad API key scoped to the target org.
   - `MONAD_CT_BUCKET` — the CloudTrail S3 bucket (`env:MONAD_CT_BUCKET`).
   - `MONAD_CT_ROLE_ARN` — the cross-account IAM role ARN (`env:MONAD_CT_ROLE_ARN`).
   - One more secret per additional `env:VAR` you reference in `monad/**`.
2. **Point at your org.** Edit `instance` / `organization_id` in `.monad-lock.json`.
   To demo create-from-scratch against an **empty** org instead of adopting the
   existing `jfrog-417c` objects, blank out every `id` (set to `""`) so the first
   apply creates them.
3. **Merge gate** on `main` — the `protect-main` ruleset (Settings → Rules →
   Rulesets) requires a PR approved by someone other than the author
   (`required_approving_review_count: 1` + `require_last_push_approval`), blocks
   force-pushes and deletion, and lists a **deploy key** as the only bypass
   actor. The apply job checks out over that write-enabled deploy key
   (`RECONCILER_SSH_KEY` secret), so its lockfile commit pushes to `main` past
   the review rule while human pushes still require a reviewed PR.
4. **Reconciler runtime.** The workflows install the Claude Code CLI
   (`npm install -g @anthropic-ai/claude-code`) and run it headless with
   `claude -p … --dangerously-skip-permissions --output-format json` on an
   ephemeral runner. (The interactive `anthropics/claude-code-action` is not
   used — it only handles @claude-mention / PR events and rejects `push`.)

## Try it

- **Change a transform:** edit `monad/transforms/drop-low-value-fields.json`,
  open a PR → the plan comment shows `UPDATE transform:drop-low-value-fields`.
  Merge → apply patches it in Monad.
- **Add an output:** copy `examples/elasticsearch-output.json` into
  `monad/outputs/`, add its secrets to GitHub + the `env:` block in `apply.yml`,
  wire it into `monad/pipelines/cloudtrail.json`.
- **Prune:** delete a file → plan shows `PRUNE …` → merge deletes it in Monad.
- **Self-heal:** delete the pipeline/input in the Monad UI (files stay in the
  repo), then run **apply** — Actions → `monad-apply` → *Run workflow*, or merge
  any change touching `monad/**`. The reconciler GETs the stale lockfile ids,
  sees they're gone, recreates the objects, and writes the new ids back to the
  lockfile. Live state is the source of truth; the repo puts it back.
