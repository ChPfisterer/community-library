# Branch protection for `main`

Use one of the options below to protect `main` so changes land through small, reviewed PRs with green CI.

## Option A — GitHub UI (quick)
1) Settings → Branches → Branch protection rules → Add rule
2) Branch name pattern: `main`
3) Check these boxes:
   - Require a pull request before merging
     - Require approvals: 1 (or more)
     - Require review from Code Owners
     - Dismiss stale pull request approvals when new commits are pushed
     - Require conversation resolution before merging
   - Require status checks to pass before merging
     - Require branches to be up to date before merging
     - Select checks: CI - API, CI - Web, CI - iOS, CI - Infra, CI - Docs
   - Require linear history (recommended with squash merge)
   - Do not allow bypassing the above settings
   - Restrict who can push to matching branches (optional; rely on PRs only)
   - Include administrators (recommended)
4) Save changes

Note: If the list of status checks shows different names (e.g., job-specific), select the exact ones you see in a green PR—names must match.

## Option B — GitHub CLI (`gh`) with JSON body
Create the protection via API using `gh api` (requires `gh auth login`).

```bash
# Write JSON to a temp file (or commit as .github/branch-protection.json)
cat > /tmp/branch-protection.json <<'JSON'
{
  "required_status_checks": {
    "strict": true,
    "contexts": [
      "CI - API",
      "CI - Web",
      "CI - iOS",
      "CI - Infra",
      "CI - Docs"
    ]
  },
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": true,
    "required_approving_review_count": 1
  },
  "restrictions": null,
  "required_linear_history": true,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "required_conversation_resolution": true
}
JSON

# Apply to this repo's main branch
OWNER=ChPfisterer
REPO=com-lib
BRANCH=main

gh api \
  -X PUT \
  -H "Accept: application/vnd.github+json" \
  "/repos/$OWNER/$REPO/branches/$BRANCH/protection" \
  --input /tmp/branch-protection.json
```

Tips
- If checks are named differently in your repo (e.g., include job names), grab exact names from a green PR and replace them in `contexts`.
- You can re-run the same command to update the rule later.

## Option C — cURL (REST API)
Requires a PAT with `repo` admin scope.

```bash
TOKEN=ghp_your_token_here
OWNER=ChPfisterer
REPO=com-lib
BRANCH=main

curl -L -X PUT \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/branches/$BRANCH/protection \
  -d '{
    "required_status_checks": {
      "strict": true,
      "contexts": [
        "CI - API",
        "CI - Web",
        "CI - iOS",
        "CI - Infra",
        "CI - Docs"
      ]
    },
    "enforce_admins": true,
    "required_pull_request_reviews": {
      "dismiss_stale_reviews": true,
      "require_code_owner_reviews": true,
      "required_approving_review_count": 1
    },
    "restrictions": null,
    "required_linear_history": true,
    "allow_force_pushes": false,
    "allow_deletions": false,
    "block_creations": false,
    "required_conversation_resolution": true
  }'
```

## Notes
- We recommend enabling squash merge and disabling direct pushes to `main` (enforced by this rule).
- If you add or rename workflows, update the `contexts` list accordingly.
- For signed commits or DCO requirements, configure those separately in repo settings or via additional endpoints.
