# Branch Protection Setup

Configure the following in GitHub repository Settings → Branches:

## `stage` branch
- Require status checks to pass before merging
  - Required: `eds/preview`
- Require branches to be up to date before merging

## `main` branch
- Require a pull request before merging
  - Required approvals: 1
- Require status checks to pass before merging
  - Required: `eds/preview`
- Do not allow bypassing the above settings
