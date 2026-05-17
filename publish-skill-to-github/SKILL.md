---
name: publish-skill-to-github
description: Publish or update QoderWork skills to GitHub as public repositories with versioned releases. Use when the user wants to upload skills to GitHub, sync skill changes to an existing repo, create releases, or when any skill modification needs to be mirrored to GitHub. Handles both first-time publishing and incremental updates.
---

# Publish / Update Skills to GitHub

## Purpose

Sync local QoderWork skills to GitHub repositories. Covers both **first-time publishing** (create repo + upload) and **incremental updates** (update existing files in repo). Designed to work reliably on Windows with curl and PowerShell fallbacks.

## When to Use

- User says "upload this skill to GitHub" or "publish my skill"
- User says "update the skill on GitHub" or "sync changes to GitHub"
- User modifies a skill and wants the GitHub mirror updated
- User mentions releasing, versioning, or open-sourcing a skill

## Prerequisites

- GitHub account username
- GitHub Personal Access Token (classic) with `repo` scope
- Skills exist under `%USERPROFILE%\.qoderwork\skills\<skill-name>\`

> **Token handling**: The agent must ask the user for the token. Never persist tokens in files or chat logs. Use it only during the current operation.

---

## Windows Environment: Critical Differences

On Windows, bash one-liners fail due to quoting and path issues. Use **PowerShell** or **file-based JSON** instead of inline JSON strings.

### Base64 Encoding (Windows)

Windows has no `base64 -w 0`. Use PowerShell:

```powershell
$bytes = [IO.File]::ReadAllBytes("C:\path\to\file.md")
$b64 = [Convert]::ToBase64String($bytes)
```

Or for a quick inline command:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('SKILL.md')) | Set-Clipboard
```

### JSON Payload Best Practice (Windows)

**Never** use inline JSON with curl on Windows cmd. Instead:

1. Write JSON to a temp file
2. Use `curl -d @temp.json`

Example for creating a repo:

```powershell
$body = @{ name = "xms-login"; private = $false; description = "QoderWork skill: xms-login" } | ConvertTo-Json -Compress
$body | Out-File -Encoding utf8 "C:\temp\repo.json" -NoNewline
curl -s -X POST "https://api.github.com/user/repos" `
  -H "Authorization: token $TOKEN" `
  -H "Accept: application/vnd.github.v3+json" `
  -H "Content-Type: application/json" `
  -d "@C:\temp\repo.json"
```

### GitHub API via PowerShell (Recommended on Windows)

For complex operations, use `Invoke-RestMethod` instead of curl:

```powershell
$headers = @{
  Authorization = "token $TOKEN"
  Accept = "application/vnd.github.v3+json"
}

# Create repo
$body = @{ name = "xms-login"; private = $false } | ConvertTo-Json -Compress
Invoke-RestMethod -Uri "https://api.github.com/user/repos" -Method POST -Headers $headers -Body $body

# Upload file
$content = [Convert]::ToBase64String([IO.File]::ReadAllBytes("SKILL.md"))
$body = @{ message = "Update SKILL.md"; content = $content } | ConvertTo-Json -Compress
Invoke-RestMethod -Uri "https://api.github.com/repos/$OWNER/$REPO/contents/$SKILL/SKILL.md" -Method PUT -Headers $headers -Body $body
```

---

## Workflow A: First-Time Publish

### Step 1: Verify local skill

```powershell
$skill = "xms-login"
$skillDir = "$env:USERPROFILE\.qoderwork\skills\$skill"
Test-Path "$skillDir\SKILL.md"  # Must be true
```

### Step 2: Check if repo already exists

```powershell
$owner = "xuluoforcainiao"
$repo = $skill
$TOKEN = "ghp_xxxxxxxx"  # Ask user for this

$response = Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$repo" -Method GET -Headers @{ Authorization = "token $TOKEN" } -ErrorAction SilentlyContinue
if ($response) {
  Write-Host "Repo already exists. Use Workflow B (Update) instead."
  return
}
```

### Step 3: Create repository

```powershell
$body = @{
  name = $repo
  private = $false
  description = "QoderWork skill: $skill"
} | ConvertTo-Json -Compress

Invoke-RestMethod -Uri "https://api.github.com/user/repos" -Method POST `
  -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.v3+json" } `
  -Body $body
```

> If 422 Unprocessable Entity: repo already exists. Proceed to Workflow B.

### Step 4: Upload all skill files

```powershell
$skillDir = "$env:USERPROFILE\.qoderwork\skills\$skill"
$files = Get-ChildItem -Path $skillDir -Recurse -File

foreach ($file in $files) {
  $relPath = $file.FullName.Substring($skillDir.Length + 1).Replace("\", "/")
  $repoPath = "$skill/$relPath"
  $content = [Convert]::ToBase64String([IO.File]::ReadAllBytes($file.FullName))

  $body = @{
    message = "Add $relPath"
    content = $content
  } | ConvertTo-Json -Compress

  try {
    Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$repo/contents/$repoPath" -Method PUT `
      -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.v3+json" } `
      -Body $body
    Write-Host "Uploaded: $repoPath"
  } catch {
    Write-Host "FAILED: $repoPath - $($_.Exception.Message)"
  }

  Start-Sleep -Milliseconds 500  # Rate limit buffer
}
```

> **Critical**: The file path inside the repo MUST be `$skill/$relPath` (e.g., `xms-login/SKILL.md`), NOT just `SKILL.md`. GitHub skill discovery expects the nested directory structure.

### Step 5: Add topics (best-effort)

```powershell
$body = @{ names = @("agent-skills") } | ConvertTo-Json -Compress

try {
  Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$repo/topics" -Method PUT `
    -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.mercy-preview+json" } `
    -Body $body
} catch {
  Write-Host "Topics update failed (non-critical): $($_.Exception.Message)"
}
```

> Topics API may return 404 for newly created repos. This is non-critical; retry after 5 seconds if desired.

### Step 6: Create release

```powershell
$body = @{
  tag_name = "v1.0.0"
  target_commitish = "main"
  name = "v1.0.0"
  body = "Initial release of $skill"
  draft = $false
  prerelease = $false
} | ConvertTo-Json -Compress

Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$repo/releases" -Method POST `
  -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.v3+json" } `
  -Body $body
```

---

## Workflow B: Update Existing Repo

Use this when the repo already exists and you need to sync modified files.

### Step 1: Identify changed files

Compare local files against GitHub. For simplicity, upload all files (idempotent if content matches):

```powershell
$skill = "xms-login"
$owner = "xuluoforcainiao"
$repo = $skill
$skillDir = "$env:USERPROFILE\.qoderwork\skills\$skill"
```

### Step 2: Upload with SHA (update) or create

For existing files, GitHub requires the file's current SHA. If you don't have it, use this strategy:

**Strategy: Try PUT without SHA first. If 422, fetch SHA and retry.**

```powershell
function Upload-File($localPath, $repoPath) {
  $content = [Convert]::ToBase64String([IO.File]::ReadAllBytes($localPath))

  # Attempt 1: Create or update without SHA (works for new files)
  $body = @{
    message = "Update $repoPath"
    content = $content
  } | ConvertTo-Json -Compress

  try {
    Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$repo/contents/$repoPath" -Method PUT `
      -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.v3+json" } `
      -Body $body
    return
  } catch {
    $err = $_.Exception.Message
    if ($err -like "*422*") {
      # File exists but SHA mismatch or required. Fetch SHA.
      try {
        $fileInfo = Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$repo/contents/$repoPath" -Method GET `
          -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.v3+json" }
        $sha = $fileInfo.sha

        # Attempt 2: Update with SHA
        $body = @{
          message = "Update $repoPath"
          content = $content
          sha = $sha
        } | ConvertTo-Json -Compress

        Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$repo/contents/$repoPath" -Method PUT `
          -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.v3+json" } `
          -Body $body
        return
      } catch {
        Write-Host "Final failure for $repoPath : $($_.Exception.Message)"
      }
    } else {
      Write-Host "Upload failed for $repoPath : $err"
    }
  }
}
```

### Step 3: Batch upload all files

```powershell
$files = Get-ChildItem -Path $skillDir -Recurse -File
foreach ($file in $files) {
  $relPath = $file.FullName.Substring($skillDir.Length + 1).Replace("\", "/")
  $repoPath = "$skill/$relPath"
  Upload-File $file.FullName $repoPath
  Start-Sleep -Milliseconds 300
}
```

---

## Workflow C: Batch Publish Multiple Skills

```powershell
$skills = @("xms-export-automation", "xms-login", "xms-browser-interaction", "xms-environment-compatibility")
$owner = "xuluoforcainiao"

foreach ($skill in $skills) {
  Write-Host "==================== $skill ===================="

  # Check if repo exists
  $exists = $false
  try {
    $null = Invoke-RestMethod -Uri "https://api.github.com/repos/$owner/$skill" -Method GET `
      -Headers @{ Authorization = "token $TOKEN" } -ErrorAction Stop
    $exists = $true
    Write-Host "Repo exists, updating..."
  } catch {
    Write-Host "Repo does not exist, creating..."
  }

  if (-not $exists) {
    # Create repo
    $body = @{ name = $skill; private = $false; description = "QoderWork skill: $skill" } | ConvertTo-Json -Compress
    Invoke-RestMethod -Uri "https://api.github.com/user/repos" -Method POST `
      -Headers @{ Authorization = "token $TOKEN"; Accept = "application/vnd.github.v3+json" } `
      -Body $body
    Start-Sleep -Seconds 2
  }

  # Upload all files (using Workflow B logic)
  $skillDir = "$env:USERPROFILE\.qoderwork\skills\$skill"
  $files = Get-ChildItem -Path $skillDir -Recurse -File
  foreach ($file in $files) {
    $relPath = $file.FullName.Substring($skillDir.Length + 1).Replace("\", "/")
    $repoPath = "$skill/$relPath"
    # ... call Upload-File function ...
    Start-Sleep -Milliseconds 300
  }

  Write-Host "DONE: $skill"
}
```

---

## Sensitive File Exclusions

Before uploading, exclude files that should never be public:

| Pattern | Reason |
|---------|--------|
| `credentials.json` | API keys |
| `*.key`, `*.pem` | Private keys |
| `.env` | Environment variables |
| `heartbeat-state.json` | Runtime state |
| `config.json` with secrets | Credentials |

Add to the upload loop:

```powershell
$excludePatterns = @("credentials.json", "heartbeat-state.json", ".env")
if ($excludePatterns -contains $file.Name) {
  Write-Host "Skipping sensitive file: $($file.Name)"
  continue
}
```

---

## Common Errors & Recovery

| Error | Cause | Fix |
|-------|-------|-----|
| `422 Unprocessable Entity` on file upload | File already exists and SHA not provided | Fetch SHA via GET, then retry PUT with sha field |
| `404 Not Found` on topics API | Repo too new, topics endpoint not ready | Retry after 5s, or skip (non-critical) |
| `401 Unauthorized` | Token invalid or expired | Ask user for a new token |
| `403 Rate Limit` | Too many requests | Wait 60 seconds and retry |
| JSON parse error in curl | Windows quoting issues | Use PowerShell `ConvertTo-Json` + file-based `-d @file` |
| Base64 line wrapping | Used `certutil` instead of PowerShell | Use `[Convert]::ToBase64String` which never wraps |

---

## Anti-Patterns

- Never commit GitHub tokens into repositories
- Never use inline JSON with curl on Windows cmd
- Never upload `credentials.json` or `.env` files
- Never place files at repo root; always use nested `repo-name/skill-name/` structure
- Do not use `base64` command on Windows; it behaves differently from Linux

---

## Post-Update Verification

After uploading, verify key files are accessible:

```powershell
$urls = @(
  "https://api.github.com/repos/$owner/$skill/contents/$skill/SKILL.md",
  "https://api.github.com/repos/$owner/$skill/contents/$skill/CHANGELOG.md"
)
foreach ($url in $urls) {
  try {
    $resp = Invoke-RestMethod -Uri $url -Method GET -Headers @{ Authorization = "token $TOKEN" }
    Write-Host "OK: $($resp.name) ($($resp.size) bytes)"
  } catch {
    Write-Host "MISSING: $url"
  }
}
```
