# HiddenLayer GHA Test Harness

A throwaway-repo template for ad-hoc troubleshooting of the
[`hiddenlayer-model-scan-github-action`](https://github.com/hiddenlayerai/hiddenlayer-model-scan-github-action).

Use this when you need to answer **"does the proposed YAML actually work end-to-end?"**.
For "why does it fail at the code level?", use the local Python repros in `../`
(`repro.py`, `repro_hf_uri.py`, `check_hf_access.py`).

## Why a throwaway repo at all?

Running the GHA itself from a real GitHub Actions runner gives you:

- The exact same code path the customer hits.
- Cloud-to-cloud bandwidth (HF CDN → GitHub runner → HL S3). Roughly 10–100× faster
  than uploading 10–25 GB models from a laptop.
- Results that automatically land in the HL console with a real `scan_id`,
  and a GitHub Actions run page you can link to in a Jira / Salesforce case.
- No Python env, no SDK version drift, no local disk pressure.

Rule of thumb: **for "does this fix work?" use this harness; for "why does this fail at line X?" use the local Python repros.**

## One-time setup (~5 minutes)

### 1. Create the throwaway repo

```
cd ~/modelScanner-AIDR
cp -R local-repro/gha-test-harness hl-gha-troubleshoot
cd hl-gha-troubleshoot
git init -b main
git add .
git commit -m "Initial harness"
gh repo create hl-gha-troubleshoot --private --source=. --push
```

Keep it private — the workflows reference your HL and HF secrets.

### 2. Add repository secrets

In the repo, go to **Settings → Secrets and variables → Actions → New repository secret**, and add:

| Secret | Value |
|---|---|
| `HL_CLIENT_ID` | Your HiddenLayer SaaS API client id |
| `HL_CLIENT_SECRET` | Your HiddenLayer SaaS API client secret |
| `HUGGINGFACE_TOKEN` | Your HF read token (must own a license on every gated repo you'll scan) |

Or with the gh CLI:

```
gh secret set HL_CLIENT_ID
gh secret set HL_CLIENT_SECRET
gh secret set HUGGINGFACE_TOKEN
```

### 3. Smoke test

Verify GitHub has indexed the workflows, then run the public flair smoke test:

```
gh workflow list
gh workflow run scan-quick-public.yml -f use_hf_path=true
gh run watch
```

(If `gh workflow list` is empty just-after-push, give GitHub ~15 seconds to index
and re-try. Always dispatch by **filename**, not display name — `gh workflow run`
is strict about display-name matching.)

If the run finishes successfully (~1 minute) and shows a row in the HL Supply
Chain console for `flair/ner-english-ontonotes`, the harness is good to go.

### A note on action versions

All workflows hardcode `@v1.0.8` of `hiddenlayerai/hiddenlayer-model-scan-github-action`.
GitHub Actions does not allow `${{ inputs.* }}` expressions in the `uses:` clause,
so we can't parameterize the version at dispatch time. To re-test against a
different release (e.g. `v1.0.5`), edit the tag in the YAML and push.

## What's in here

| Workflow file | Purpose | Trigger |
|---|---|---|
| `.github/workflows/scan.yml` | Parameterized generic harness. Pick model_path, gha_version, community_scan, runner disk-cleanup, etc. from the Actions UI. | `workflow_dispatch` (with inputs) |
| `.github/workflows/scan-quick-public.yml` | 30–60 sec smoke test against `flair/ner-english-ontonotes`. Verifies HL & action wiring. | `workflow_dispatch` |
| `.github/workflows/scan-fix-gemma-3-12b.yml` | The customer's gated model via Andrew's `hf://` workaround (the working fix). | `workflow_dispatch` |
| `.github/workflows/scan-broken-community.yml` | Reproduces the customer's failing config (`community_scan: HUGGING_FACE` on a gated repo). Expected outcome: backend returns `HUGGING_FACE_REPO_GATED`. | `workflow_dispatch` |

## Common dispatch commands

Always dispatch by filename. Display names with spaces / special characters
fail to match in `gh workflow run`.

```
gh workflow run scan-quick-public.yml -f use_hf_path=true

gh workflow run scan-fix-gemma-3-12b.yml

gh workflow run scan-broken-community.yml

gh workflow run scan.yml \
  -f model_path="hf://meta-llama/Llama-3.2-1B" \
  -f free_disk_space=false \
  -f fail_on_detection=false
```

Then `gh run watch` (or `gh run view --log`) to follow.

## Decision matrix: which scan path to test?

| You want to test... | model_path | community_scan | HF token used? | Notes |
|---|---|---|---|---|
| The Community Scan endpoint on a public HF repo | `flair/ner-english-ontonotes` | `HUGGING_FACE` | no | Should succeed |
| The Community Scan endpoint on a **gated** HF repo | `google/gemma-3-12b-it` | `HUGGING_FACE` | no (irrelevant — community_scan ignores it) | Expected to fail with `HUGGING_FACE_REPO_GATED` — this is the customer's bug |
| The `hf://` path on a public HF repo | `hf://flair/ner-english-ontonotes` | _(blank)_ | optional | Should succeed, no token needed |
| The `hf://` path on a **gated** HF repo (Andrew's fix) | `hf://google/gemma-3-12b-it` | _(blank)_ | yes (`HUGGINGFACE_TOKEN` env) | Should succeed; this is the customer's workaround |
| A model on S3 | `s3://my-bucket/path/to/model.bin` | _(blank)_ | no | Needs `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` secrets |
| A model in the repo itself | `./models/pytorch_model.bin` | _(blank)_ | no | Useful for testing arbitrary uploaded models |

## Disk-space cheat sheet for gated HF models

GitHub's standard `ubuntu-latest` has ~14 GB free. For larger models you'll get
`ENOSPC` mid-download.

| Model | Approx size | Standard runner OK? | Workarounds |
|---|---|---|---|
| `flair/ner-english-ontonotes` | ~0.6 GB | yes | none needed |
| `google/gemma-2-2b` | ~5 GB | yes | none needed |
| `meta-llama/Llama-3.2-1B` | ~2.5 GB | yes | needs gated-license approval first |
| `google/gemma-3-12b-it` | ~24 GB | **no** | use `free_disk_space=true` (frees ~30 GB) or a `ubuntu-latest-16-core` runner |
| `openai/gpt-oss-120b` | ~235 GB | no | self-hosted runner only |

## When you're done with the harness

You can delete the whole throwaway repo from GitHub (`gh repo delete hl-gha-troubleshoot --yes`)
and `rm -rf` the local copy. Nothing persistent depends on it; results live in the
HL console under your tenant.
