# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**cfn-delete-action** is a GitHub Action for deleting AWS CloudFormation stacks with real-time logging and comprehensive error handling. It's a **composite action** (not containerized) that orchestrates bash scripts running on GitHub Actions runners.

**Key Technologies**:
- Bash scripting (action logic)
- AWS CLI (CloudFormation operations)
- Node.js + npm (release automation)
- GitHub Actions (CI/CD and distribution)
- Semantic Release (automated versioning)

## Architecture

### Composite Action Structure

The action is defined in `action.yaml` with three main steps:

1. **Delete CloudFormation Stack** - Core deletion logic using AWS CLI
   - Checks if stack exists
   - Initiates deletion
   - Waits for completion with exponential backoff
   - Captures timing and status

2. **Get CloudFormation Stack Events** - Retrieves stack event logs for debugging
   - Queries CloudFormation events
   - Handles case where stack is already deleted

3. **Generate GitHub Actions Summary** - Creates human-readable report
   - Formats results into GitHub Actions summary
   - Shows stack name, region, status, duration

### Input/Output Contract

**Inputs**:
- `stack-name` (required) - CloudFormation stack name
- `aws-region` (optional, default: `us-east-1`) - AWS region

**Outputs**:
- `stack-status` - Final deletion status (SUCCESS, FAILED, STACK_NOT_FOUND)
- `deletion-time` - Duration in minutes

### Release Automation

The project uses **Semantic Release** configured in `scripts/plugins/release.config.js`:
- Analyzes commits using Conventional Commits format
- Generates CHANGELOG.md entries
- Creates GitHub releases
- Bumps version based on commit types (major/minor/patch)
- Runs automatically on `main` branch push via `.github/workflows/release.yaml`

## Common Development Tasks

### Install Dependencies
```bash
npm install
```

### Run Semantic Release Locally
```bash
GITHUB_TOKEN=<your-token> npm run release
```
Requires `GITHUB_TOKEN` with `repo` and `workflow` scopes.

### Test the Action Locally
```bash
# Set up AWS credentials
export AWS_ACCESS_KEY_ID=<key>
export AWS_SECRET_ACCESS_KEY=<secret>
export AWS_DEFAULT_REGION=us-east-1

# Test basic flow (requires existing stack for real testing)
bash -c 'source action.yaml && ...'  # Limited - action.yaml is declarative
```

### Commit Message Format
Use [Conventional Commits](https://www.conventionalcommits.org/):
- `feat: ` - New feature (minor version bump)
- `fix: ` - Bug fix (patch version bump)
- `chore: ` - Non-code changes (no version bump)
- `BREAKING CHANGE:` - Major version bump

Examples:
```
feat: Add stack retention policy support
fix: Handle DELETE_IN_PROGRESS state correctly
chore: Update README examples
```

## Key Files and Their Purpose

| File | Purpose |
|------|---------|
| `action.yaml` | Composite action definition - orchestrates all steps |
| `package.json` | npm dependencies for semantic-release |
| `scripts/plugins/release.config.js` | Semantic Release configuration |
| `.github/workflows/release.yaml` | Auto-release workflow on main push |
| `.github/workflows/claude-code-review.yaml` | Runs Claude code review on PRs |
| `README.md` | User-facing documentation with examples |
| `CHANGELOG.md` | Auto-generated release notes |

## Important Patterns

### Error Handling in action.yaml
- Uses `set -e` to fail on first error
- Checks stack existence before deletion
- Handles already-deleted stacks gracefully
- Provides detailed error messages in outputs
- Always runs "Display Summary" step with `if: always()`

### AWS API Interaction
- Uses AWS CLI v1 commands (describe-stacks, delete-stack, wait)
- Relies on credentials from environment (set by `aws-actions/configure-aws-credentials`)
- Implements exponential backoff via `wait` command for throttling
- Redirects stderr to stdout for logging visibility

### Output Handling
- Bash steps write to `$GITHUB_OUTPUT` for inter-step communication
- Summary step writes to `$GITHUB_STEP_SUMMARY` for GitHub Actions UI
- All outputs are string-based (version is "0.0.0" until first release)

## Testing the Action

There are no automated tests in this repo currently. To test changes:

1. **Locally**: Create a test stack in AWS, then modify the action and test manually
2. **In PR**: Create a test workflow in `.github/workflows/test-action.yaml` that uses the branch
3. **Release Preview**: semantic-release can be run with `--dry-run` flag (requires modification to npm script)

## Workflow Files

- **release.yaml**: Runs `npm ci` and `npm run release` on main push
- **claude-code-review.yaml**: Runs Claude code review on PR changes
- **claude.yaml**: Available for Claude-specific automation (currently minimal setup)
- **create-branch.yaml**: Helper for branch creation
- **notify.yaml**: Sends notifications (currently minimal)

## Guidelines for Changes

### Adding New Features
1. Update `action.yaml` with new inputs/outputs
2. Implement logic in appropriate bash step
3. Update README.md with examples
4. Use conventional commit message (`feat: ...`)
5. Push to main or open PR for review

### Bug Fixes
1. Identify the failing bash script or step
2. Add error handling or fix logic
3. Use conventional commit message (`fix: ...`)
4. Consider edge cases (already-deleted stacks, permission errors, timeouts)

### Documentation Updates
1. Modify README.md for user-facing docs
2. Update this file (CLAUDE.md) for developer guidance
3. Use `chore:` commit prefix

## Dependency Management

**Production**: None directly in the action (relies on AWS CLI on runner)
**Development**: Semantic Release ecosystem
- `@semantic-release/*` packages - Release automation plugins
- `commitizen` - Enforces commit format
- `cz-conventional-changelog` - Changelog generator

## AWS Permissions Required

Users of this action must configure AWS credentials with these CloudFormation permissions:
```json
{
  "Action": [
    "cloudformation:DeleteStack",
    "cloudformation:DescribeStacks",
    "cloudformation:DescribeStackEvents",
    "cloudformation:DescribeStackResources"
  ],
  "Resource": "*"
}
```

This is documented in README.md for end users.
