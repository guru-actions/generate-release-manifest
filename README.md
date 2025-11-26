# generate-release-manifest

This action queries the **CloudBees API** and produces a **release-style SquidStack manifest** suitable for the “Deploy CB Squidstack (release-style)” workflow.

---

## 🚀 What this action does

- Reads a list of components (via `components_json`)
- For each component:
  - Calls `/v3/components/{id}/artifactinfos`
  - Optionally filters artifacts using label selectors
  - Selects the “latest” artifact (by `publishedOn` if available)
  - Optionally resolves human-friendly component names (`org_id` → org-scoped lookup)
- Generates a **release-style manifest**, identical in structure to a CloudBees Release manifest (per-component `deploy` + nested repo blocks).
- Provides detailed **timing statistics** for performance debugging.

---

## 📥 Inputs

### `cb_api_url` (required)
Base URL for CloudBees API (e.g. `https://api.cloudbees.io`).

### `cb_pat` (required)
CloudBees Personal Access Token (Bearer auth).

### `org_id` (optional)
Organization ID.  
Used only for optional component-name resolution.

### `labels` (optional)
Comma-separated filter list, e.g.:

```text
rel=squid,ns=squid-demo-3
```

Each entry must match an artifact’s `labels` array (each stored as `"key=value"`).

If omitted, no filtering is applied and all artifacts are considered.

### `components_json` (required)

JSON object describing which components to resolve and how. Example:

```json
{
  "kraken-auth": {
    "component_id": "2dc859d8-f3e1-44ca-9160-6e86a8077aab",
    "repo": "gururepservice/kraken-auth"
  },
  "squid-ui": {
    "component_id": "0459d9f1-4f37-4db1-bfb4-e2e1731a0975",
    "repo": "gururepservice/squid-ui"
  }
}
```

- **Key**: friendly component key (e.g. `"kraken-auth"`) – this becomes the top-level key in the manifest.
- **`component_id`**: UUID used for `/v3/components/{id}/artifactinfos`.
- **`repo`**: nested repo key under that component (e.g. `"gururepservice/kraken-auth"`).

You can add extra fields for your own bookkeeping; the action only requires `component_id` and `repo`.

### `debug` (optional)

Controls verbosity of logging to stderr. One of:

- `"info"` (default): brief info + summary
- `"debug"`: more chatter about resolution
- `"timer"`: focused on timings (`artifactinfos`/`components` calls and totals)

---

## 📤 Outputs

### `manifest`

A **JSON** string containing the release-style manifest. Example shape:

```json
{
  "kraken-auth": {
    "deploy": true,
    "id": "2dc859d8-f3e1-44ca-9160-6e86a8077aab",
    "gururepservice/kraken-auth": {
      "deploy": true,
      "name": "gururepservice/kraken-auth",
      "url": "gururepservice/kraken-auth:latest",
      "version": "latest",
      "digest": "sha256:971ec7d48fa10dbc8f3d3c3a7f2356fa8ec06ee83421ca390027ce42c98c0d33",
      "id": "5507ef11-45fd-4c60-9464-b91c4fa47c38"
    }
  },
  "squid-ui": {
    "deploy": true,
    "id": "0459d9f1-4f37-4db1-bfb4-e2e1731a0975",
    "gururepservice/squid-ui": {
      "deploy": true,
      "name": "gururepservice/squid-ui",
      "url": "gururepservice/squid-ui:latest",
      "version": "latest",
      "digest": "sha256:f8646b1afbdc997706eb504c5e722c4a851f16ece1bf6f8c53c602aff69b4a56",
      "id": "6afce8d3-5fbf-4cd3-9e4c-9c33aba35575"
    }
  }
}
```

Notes:

- Top-level key is your logical component key from `components_json`.
- `deploy` is always `true` for the repo that matched the labels.
- You can choose later which components/repos to deploy by toggling the `deploy` flags in the manifest.

### `stats`

A **JSON** string containing timing/call information, e.g.:

```json
{
  "artifactinfos_calls": 12,
  "artifactinfos_time_s": 1.234,
  "components_calls": 4,
  "components_time_s": 0.456,
  "overall_total_time_s": 1.79
}
```

Use this to monitor performance when `debug=timer` or when you need to understand API costs.

---

## 🔧 Example: Use in a workflow

```yaml
apiVersion: automation.cloudbees.io/v1alpha1
kind: workflow
name: Build release manifest

on:
  workflow_dispatch: {}

jobs:
  generate-manifest:
    steps:
      - id: gen
        name: Generate release-style manifest
        uses: ./.cloudbees/actions/generate-release-manifest
        with:
          cb_api_url: ${{ secrets.CB_API_URL }}
          cb_pat:     ${{ secrets.CB_PAT }}
          org_id:     ${{ secrets.CB_ORG_ID }}
          labels:     "rel=squidstack,ns=squid-demo-3"
          debug:      "timer"
          components_json: |
            {
              "kraken-auth": {
                "component_id": "2dc859d8-f3e1-44ca-9160-6e86a8077aab",
                "repo": "gururepservice/kraken-auth"
              },
              "squid-ui": {
                "component_id": "0459d9f1-4f37-4db1-bfb4-e2e1731a0975",
                "repo": "gururepservice/squid-ui"
              }
            }

      - name: Show manifest
        uses: docker://alpine:3.20
        shell: sh
        run: |
          echo "=== MANIFEST (JSON) ==="
          printf '%s
' '${{ steps.gen.outputs.manifest }}' | jq .
          echo
          echo "=== STATS ==="
          printf '%s
' '${{ steps.gen.outputs.stats }}' | jq .
```

You can then feed `${{ steps.gen.outputs.manifest }}` into your **Deploy CB Squidstack (release-style)** workflow as the `manifest` input.

---

## 🧠 Behavior details

### Artifact selection

For each component in `components_json`:

1. Call:
   ```text
   GET {CB_API_URL}/v3/components/{component_id}/artifactinfos
   ```
2. Filter `.artifacts[]` with the `labels` constraint, if provided.
3. Pick the **latest** artifact (either by:
   - `publishedOn` (ISO time) descending, OR
   - fallback to plain order if that field is missing).
4. Build a nested repo block under the given `repo` key.

If **no artifact** matches the labels for a component, that component is **omitted** from the manifest and contributes to timing, but isn’t listed.

### Component naming

We don’t strictly need a human-readable name for the manifest to work, but we try to infer one in a friendly way for later use (e.g., logging). For now that is internal to the action; it doesn’t change the manifest shape beyond `name` inside the repo block (taken from artifact metadata where possible).

### Debug levels

- `debug=info`  
  Minimal logs: key calls, counts, and summary.

- `debug=debug`  
  Includes per-component decision logs (which artifact chosen, label matches, etc.).

- `debug=timer`  
  Focused on timing lines, making it easy to profile `artifactinfos`/`components` calls.

---

## 🧪 Local testing

Inside the action folder, you can shell into the same Python image to experiment:

```bash
docker run --rm -it python:3.11-alpine sh
```

Then install what the action uses:

```bash
apk add --no-cache curl jq coreutils
```

You can paste in snippets from `action.yaml`’s `run:` section and tweak them interactively to understand or extend the behavior.

---

## 📎 Notes

- The manifest produced is **release-style** only – it’s not the “flat” manifest with `artifact_id/version/commit_sha` at top level.
- If you later want to support both flat + release formats, you can create a tiny normalizer workflow that converts this manifest into whichever flavor your deploy pipeline expects.
- This action is intentionally **lean and scoped**: it does a single job (resolve latest artifacts → build manifest) and leaves deployment concerns to a separate workflow.
