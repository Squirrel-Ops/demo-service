# demo-service

Demo victim repository for the **"SHA Pinning Is Not Enough"** supply chain attack.

> Blog post: <https://rosesecurity.dev/2026/03/24/sha-pinning-is-not-enough.html>

This repo reproduces the Trivy-style attack against `yc-actions/yc-iam-token@v1`.

## The setup

| Piece | Trivy attack (real) | This demo |
|---|---|---|
| Upstream action | `actions/checkout` | `yc-actions/yc-iam-token` |
| Clean pin | `actions/checkout@de0fac2e… # v6.0.2` | `yc-actions/yc-iam-token@5966c246… # v1` (real `v1` commit) |
| Malicious commit | `70379aad…` — orphaned commit reachable from `actions/checkout` via fork network | `f4daf197…` — orphaned commit reachable from `yc-actions/yc-iam-token` via fork `Squirrel-Ops/yc-iam-token` |
| Payload | Composite action: legit checkout + silent downloads | Composite action: base64-encodes the secrets and prints them instead of authenticating |
| Delivery commit | `fix(ci): Use correct checkout pinning` (14 lines, mostly quote/whitespace noise) | `fix(ci): Use correct pinning for yc-iam-token` (same pattern) |
| The lie | `# v6.0.2` comment kept while SHA changed | `# v1` comment kept while SHA changed |

## How to walk the demo

1. **Initial state** — the first commit pins the action to the *real* `v1` commit with the `# v1` comment.
2. **The attack** — the second commit (`fix(ci): Use correct pinning for yc-iam-token`) buries the SHA swap inside quote-style noise. The `# v1` comment never changes, so a reviewer's eye skips the hex string — exactly what happened in Trivy.
3. **The payload** — the SHA resolves fine (GitHub Actions doesn't warn about orphaned commits), and the action runs a composite step that prints the base64-encoded `YC_SA_JSON_CREDENTIALS` secret:
   ```
   IAM token issued (188 chars):
   eyJpZCI6ImFqZXRlc3QxMjM0NTY3ODkwYWJjZCIsInNlcnZpY2VfYWNjb3VudF9p
   ...
   ```
   Decode it and you get the "stolen" secret. The `token` output is still set, so downstream steps keep working — the backdoor is silent.

## Where the backdoor lives

The commit `f4daf19710c2b773d90c1a2b5fa7cb3750f315fb` belongs to **no branch anywhere** — it was pushed to a temp ref in the fork `Squirrel-Ops/yc-iam-token` and the ref was deleted. You can verify with the check the blog recommends:

```
gh api repos/yc-actions/yc-iam-token/commits/f4daf19710c2b773d90c1a2b5fa7cb3750f315fb/branches-where-head
# → []
```

An empty list = orphaned commit = the yellow "does not belong to any branch" banner. This is the floor SHA pinning gives you: it guarantees the *same* commit, not a commit that was ever part of the upstream project.

## Lessons (from the blog)

- Pin SHAs — yes — but **verify what they point to** before merging a SHA change.
- A SHA must resolve to a commit on a real release branch/tag of the upstream repo, not a fork orphan.
- The version comment is free text. Nobody reads hex strings; the comment is what reviewers actually read.
- Belt and braces: signed commits on workflow changes, org-level action allowlists, mirrored actions, artifact attestations.
