---
name: publish-skill-to-github
description: Publish QoderWork skills to GitHub as versioned releases for public sharing and team distribution. Use when the user wants to upload a skill to GitHub, share a skill with colleagues, create a skill release, bundle a local skill for distribution, or publish an agent skill to the open ecosystem.
---

# Publish Skill to GitHub

## Purpose

Package a local QoderWork skill and publish it to GitHub as a public repository with a versioned release, so others can install it via the Skills CLI or manual download.

## When to Use

- User says "upload this skill to GitHub" or "publish my skill"
- User wants to share a skill with teammates or the community
- User asks how to distribute a skill outside of QoderWork
- User mentions releasing, versioning, or open-sourcing a skill

## Prerequisites

- Git installed and available in PATH
- GitHub account with a Personal Access Token (classic) that has `repo` scope
- The skill already exists under `~/.qoderwork/skills/<skill-name>/`

## Workflow

### Step 1: Identify the Skill

Determine the skill name and verify the local directory:

```bash
ls ~/.qoderwork/skills/<skill-name>/
```

Expected contents include at minimum `SKILL.md`. Optional files: `reference.md`, `examples.md`, `scripts/`.

### Step 2: Prepare Release Directory

Create a clean staging directory with the correct nested structure (`repo-name/skill-name/`):

```bash
SKILL="<skill-name>"
OWNER="<github-username>"
TOKEN="<github-pat>"
SKILLS_BASE="$HOME/.qoderwork/skills"
TEMP_BASE="$HOME/.qoderwork/workspace"
SKILL_DIR="$SKILLS_BASE/$SKILL"
RELEASE_DIR="$TEMP_BASE/$SKILL-release"
REPO_SUBDIR="$RELEASE_DIR/$SKILL"

rm -rf "$RELEASE_DIR"
mkdir -p "$REPO_SUBDIR"
cp "$SKILL_DIR"/* "$REPO_SUBDIR/"
```

> On Windows, use `%USERPROFILE%\.qoderwork\skills` and appropriate path separators, or run the commands inside Git Bash / WSL.

### Step 3: Create GitHub Repository

If `gh` CLI is available:

```bash
cd "$RELEASE_DIR"
gh repo create "$OWNER/$SKILL" --public --source=. --remote=origin --push
```

If `gh` is not available, use the GitHub API via curl:

```bash
curl -s -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/user/repos \
  -d "{\"name\":\"$SKILL\",\"private\":false,\"description\":\"QoderWork skill: $SKILL\",\"topics\":[\"agent-skills\"]}"
```

Then initialize git and push:

```bash
cd "$RELEASE_DIR"
git init
git add .
git commit -m "Initial commit: $SKILL skill"
git branch -M main
git remote add origin "https://$TOKEN@github.com/$OWNER/$SKILL.git"
git push -u origin main
```

### Step 4: Create a Release

Tag the commit and create a GitHub release (v1.0.0 as the initial version):

With `gh`:
```bash
gh release create v1.0.0 --title "v1.0.0 - Initial release" --notes "Initial release of $SKILL"
```

Without `gh`, via API:
```bash
curl -s -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/$OWNER/$SKILL/releases" \
  -d "{\"tag_name\":\"v1.0.0\",\"name\":\"v1.0.0 - Initial release\",\"body\":\"Initial release of $SKILL\"}"
```

### Step 5: Verify and Share

Confirm the repository is public and accessible at:

```
https://github.com/$OWNER/$SKILL
```

Share the install command with others:

```bash
npx skills add $OWNER/$SKILL@$SKILL
```

Or for QoderWork users, provide the raw `SKILL.md` URL for manual installation.

## Anti-Patterns

- Do not commit the GitHub token into the repository
- Do not use backslash paths in scripts; always use forward slashes or platform-appropriate wrappers
- Do not skip the nested directory structure (`repo/skill-name/SKILL.md`) unless the target ecosystem expects a flat layout

## Troubleshooting

**Push rejected (repository not found)**: Verify the repo was created successfully and the token has `repo` scope.
**Permission denied**: Ensure the token is a classic PAT with full repo access, not a fine-grained token, if using the curl API approach.
**Topic not showing**: The `topics` field via API may require a separate follow-up request to set topics on the repository.
