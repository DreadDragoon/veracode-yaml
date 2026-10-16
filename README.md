# Veracode Workflow App: Non-Blocking Config

A `veracode.yml` for the Veracode GitHub Workflow App that runs scans in **report-only mode**, so findings and scan errors never fail a build or block a merge. It also cuts down on duplicate scans so feedback comes back faster.

This is based on the `veracode.yml` from [ancient-cthulhu/Workflow-App-Bulk-Enterprise-Integration](https://github.com/ancient-cthulhu/Workflow-App-Bulk-Enterprise-Integration), and it gets rolled out to every org with that repo's `script.py`.

## Why

We want Veracode running everywhere so we can see what's out there, but we're not ready to gate development on it yet. Once teams have a handle on the findings, we can flip the break flags back on one scan type at a time.

## What changed from the bulk-integration default

### Non-blocking

| Scan | Setting | Before | After |
|---|---|---|---|
| SCA | `break_build_on_error` | `true` | `false` |
| SCA | `break_build_policy_findings` | not set (documented default is `true`) | `false` |
| IaC / secrets | `break_build_policy_findings` | `true` | `false` |
| Static | all break flags | already `false` | unchanged |

### Faster

| Scan | Setting | Before | After | Why |
|---|---|---|---|---|
| SCA | `push.branches_to_run` | `'*'` | `default_branch` | Feature branches were scanned on push and again on PR sync |
| SCA | `pull_request.action` | `opened`, `synchronize` | `opened` | Dependencies rarely change between PR commits |
| IaC / secrets | `push.branches_to_run` | `'*'` | `default_branch` | Same double-scan issue as SCA |

### Visibility

| Scan | Setting | Before | After |
|---|---|---|---|
| Static | `create_code_scanning_alert` | `false` | `true` |

Findings still show up in the check run summary and now also in each repo's **Security > Code scanning** tab, so nothing gets lost just because it isn't blocking.

### Left as-is

- `analysis_on_platform: true` for static, so results still land on the Veracode Platform
- Static push scans run on every branch except the default branch
- Issue commands, including `Veracode All Scans` for on-demand runs

## Rolling it out

Requires a classic PAT with `admin:org`, `repo` and `read:enterprise`. No Veracode credentials are needed for this step.

```bash
export GITHUB_TOKEN="..."

# Dry run: shows which orgs would be updated
python script.py --enterprise YOUR-ENTERPRISE \
  --update-veracode-yml /path/to/veracode.yml

# Apply
python script.py --apply --enterprise YOUR-ENTERPRISE \
  --update-veracode-yml /path/to/veracode.yml
```

The script backs up each org's current file as `default-veracode.yml` before overwriting it, so rolling back is a matter of restoring that file. Orgs where the `veracode` repo is missing or not fully imported get skipped with a warning.

To limit the rollout to a few orgs first, add `--orgs-file pilot-orgs.txt` (one org per line).

## Overriding per repo

A repo can override any of these settings by adding its own `veracode.yml` at the repo root with just the keys it wants to change. For example, a team that's ready to enforce policy on static findings:

```yaml
veracode_static_scan:
  break_build_policy_findings: true
```

## Things to keep in mind

- **Don't make the Veracode checks required** in branch protection or rulesets while running in this mode. A required check that's slow or stuck will still hold up merges, no matter what the break flags say.
- **Static is now the main source of scan volume.** It still runs on every push to every non-default branch. If scans are still too slow, the next step is moving static to PR-triggered only, but that changes when developers get feedback.
- **Java builds are the slowest part.** By default the app compiles the project in its own packager container. Repos that already build an artifact in Actions can skip that with `use_custom_workflow` and `artifact_file`. See [Use your source repositories' build steps](https://docs.veracode.com/r/GitHub_Workflow_Integration_for_Repo_Scanning#use-your-source-repositories-build-steps).

## Going back to enforcing

When we're ready, turn things back on gradually rather than all at once:

1. IaC / secrets: set `break_build_policy_findings: true` (secrets in code are the easiest to fix and the most urgent)
2. Static: set `break_build_policy_findings: true`
3. SCA: set `break_build_policy_findings: true`
4. Make the checks required in rulesets

Then push the updated file with `--update-veracode-yml` again.

## References

- [Veracode GitHub Workflow Integration docs](https://docs.veracode.com/r/GitHub_Workflow_Integration_for_Repo_Scanning)
- [Workflow-App-Bulk-Enterprise-Integration](https://github.com/ancient-cthulhu/Workflow-App-Bulk-Enterprise-Integration)
