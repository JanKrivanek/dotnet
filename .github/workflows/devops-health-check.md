---
name: "DevOps Daily Health Check"
description: >
  Orchestrator workflow that collects Azure DevOps and GitHub health signals,
  computes a fingerprint-based diff against the previous run, updates a pinned
  health dashboard issue, and dispatches investigation workers for new
  critical/warning findings. Adapted for the dotnet/dotnet VMR which uses
  Azure DevOps pipelines as the primary CI system.

on:
  schedule:
    - cron: "0 5 * * *"  # 05:00 UTC daily
  workflow_dispatch:

  steps:
    - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      name: Checkout config and action files
      with:
        persist-credentials: false
        sparse-checkout: |
          .github/devops-health-config.json
          .github/actions/select-copilot-pat
          src/source-manifest.json
        sparse-checkout-cone-mode: false
        fetch-depth: 1

    - id: select-copilot-pat
      name: Select Copilot token from pool
      uses: ./.github/actions/select-copilot-pat
      env:
        SECRET_0: ${{ secrets.COPILOT_GITHUB_TOKEN }}

    - name: Collect health data from Azure DevOps and GitHub
      env:
        AZURE_DEVOPS_PAT: ${{ secrets.AZURE_DEVOPS_PAT }}
        GH_TOKEN: ${{ github.token }}
      run: |
        set -euo pipefail
        mkdir -p health-data
        CONFIG=".github/devops-health-config.json"
        ERRORS=0

        ado_get() {
          local org="$1" project="$2" path="$3" out="$4"
          local url="https://dev.azure.com/${org}/${project}/_apis${path}"
          if [ -n "${AZURE_DEVOPS_PAT:-}" ]; then
            if ! curl -sf --retry 2 --max-time 30 -u ":${AZURE_DEVOPS_PAT}" "$url" -o "$out" 2>/dev/null; then
              echo "{\"error\": true, \"url\": \"$url\"}" > "$out"
              ERRORS=$((ERRORS + 1))
            fi
          else
            # Try unauthenticated for public org
            if ! curl -sf --retry 2 --max-time 30 "$url" -o "$out" 2>/dev/null; then
              echo "{\"error\": true, \"url\": \"$url\"}" > "$out"
              ERRORS=$((ERRORS + 1))
            fi
          fi
        }

        NOW=$(date -u +%Y-%m-%dT%H:%M:%SZ)
        AGO_24H=$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)
        AGO_14D=$(date -u -d '14 days ago' +%Y-%m-%dT%H:%M:%SZ)

        # ── P1/P2/P3/P4/P6: Azure DevOps build data ──────────────────────
        for org in $(jq -r '.azure_devops.organizations[].name' "$CONFIG"); do
          project=$(jq -r --arg o "$org" '.azure_devops.organizations[] | select(.name==$o) | .project' "$CONFIG")
          requires_auth=$(jq -r --arg o "$org" '.azure_devops.organizations[] | select(.name==$o) | .requires_auth' "$CONFIG")

          # Skip internal org if no PAT available
          if [ "$requires_auth" = "true" ] && [ -z "${AZURE_DEVOPS_PAT:-}" ]; then
            echo "Skipping $org (requires auth, no PAT)"
            continue
          fi

          for def_id in $(jq -r --arg o "$org" '.azure_devops.organizations[] | select(.name==$o) | .definitions[].id' "$CONFIG"); do
            for branch in $(jq -r '.branches[]' "$CONFIG"); do
              safe_branch="${branch//\//-}"

              # Last 24h builds (for P1 failures, P2 timeouts)
              ado_get "$org" "$project" \
                "/build/builds?definitions=${def_id}&branchName=refs/heads/${branch}&statusFilter=completed&minTime=${AGO_24H}&\$top=50&api-version=7.1" \
                "health-data/builds-24h-${org}-${def_id}-${safe_branch}.json"

              # Last 14d builds (for P3 duration trend, P4 pass rate)
              ado_get "$org" "$project" \
                "/build/builds?definitions=${def_id}&branchName=refs/heads/${branch}&statusFilter=completed&minTime=${AGO_14D}&\$top=200&api-version=7.1" \
                "health-data/builds-14d-${org}-${def_id}-${safe_branch}.json"
            done

            # Fetch timeline for each FAILED build in last 24h (for P1 detail)
            for f in health-data/builds-24h-${org}-${def_id}-*.json; do
              [ -f "$f" ] || continue
              jq -r '.value[]? | select(.result=="failed") | .id' "$f" 2>/dev/null | head -5 | while read -r bid; do
                ado_get "$org" "$project" \
                  "/build/builds/${bid}/timeline?api-version=7.1" \
                  "health-data/timeline-${org}-${bid}.json"
              done
            done
          done
        done

        # ── P5: Codeflow PR status (GitHub API) ──────────────────────────
        gh api "repos/${GITHUB_REPOSITORY}/pulls?state=open&per_page=100" --jq \
          '[.[] | select(.title | test("Source code updates from")) | {number, title, created_at, updated_at, head: .head.sha, user: .user.login}]' \
          > health-data/codeflow-prs.json 2>/dev/null || echo '[]' > health-data/codeflow-prs.json

        # For each codeflow PR, fetch combined commit status
        jq -r '.[].head' health-data/codeflow-prs.json 2>/dev/null | while read -r sha; do
          gh api "repos/${GITHUB_REPOSITORY}/commits/${sha}/status" \
            > "health-data/pr-status-${sha}.json" 2>/dev/null || echo '{"state":"unknown"}' > "health-data/pr-status-${sha}.json"
        done

        # ── I1: Open operational issues (GitHub API) ─────────────────────
        gh api "repos/${GITHUB_REPOSITORY}/issues" --paginate \
          -f labels=area-unified-build-OperationalIssue -f state=open -f per_page=50 \
          > health-data/operational-issues.json 2>/dev/null || echo '[]' > health-data/operational-issues.json

        # ── I6: NuGet feed health ────────────────────────────────────────
        echo '[]' > health-data/nuget-feed-status.json
        for feed in $(jq -r '.nuget_feeds[]' "$CONFIG"); do
          status=$(curl -sf --max-time 10 -o /dev/null -w '%{http_code}' "$feed" 2>/dev/null || echo "000")
          jq --arg f "$feed" --arg s "$status" '. + [{"feed": $f, "status": ($s | tonumber)}]' \
            health-data/nuget-feed-status.json > health-data/nuget-feed-status.json.tmp
          mv health-data/nuget-feed-status.json.tmp health-data/nuget-feed-status.json
        done

        # ── I7: Source manifest ──────────────────────────────────────────
        if [ -f "src/source-manifest.json" ]; then
          cp src/source-manifest.json health-data/source-manifest.json
        else
          echo '{}' > health-data/source-manifest.json
        fi

        # ── P7: GitHub Actions failures ──────────────────────────────────
        gh api "repos/${GITHUB_REPOSITORY}/actions/runs?branch=main&status=failure&per_page=30" \
          > health-data/gha-failures.json 2>/dev/null || echo '{"workflow_runs":[]}' > health-data/gha-failures.json

        # ── Manifest ─────────────────────────────────────────────────────
        echo "{\"collected_at\": \"${NOW}\", \"errors\": ${ERRORS}}" > health-data/manifest.json
        echo "Data collection complete. Files: $(ls health-data/*.json | wc -l), Errors: ${ERRORS}"

if: ${{ !(github.event_name == 'schedule' && github.event.repository.fork) }}

jobs:
  pre-activation:
    outputs:
      copilot_pat_number: ${{ steps.select-copilot-pat.outputs.copilot_pat_number }}

engine:
  id: copilot
  env:
    COPILOT_GITHUB_TOKEN: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', secrets.COPILOT_GITHUB_TOKEN, secrets.COPILOT_GITHUB_TOKEN) }}

permissions:
  contents: read
  actions: read
  issues: read
  pull-requests: read

imports:
  - ../aw/shared/devops-health.lock.md

tools:
  github:
    toolsets: [repos, issues, actions, pull_requests]
  cache-memory:
  bash: ["cat", "grep", "head", "tail", "find", "ls", "wc", "jq", "date", "sort", "uniq", "diff"]
  edit:

safe-outputs:
  create-issue:
    max: 1
  update-issue:
    target: "*"
    max: 1
  add-comment:
    target: "*"
    max: 1
  dispatch-workflow:
    workflows:
      - devops-health-investigate
    max: 2
  noop:
    report-as-issue: false

network:
  allowed:
    - defaults

timeout-minutes: 60
---

# DevOps Daily Health Check — Orchestrator (dotnet/dotnet VMR)

You are a DevOps infrastructure health monitoring agent for the dotnet/dotnet Virtual Monolithic Repository. Your job is to analyze pre-collected pipeline and infrastructure health signals, compute a diff against the previous run, and produce a comprehensive yet actionable health dashboard.

> **Key difference from dotnet/skills:** This repo primarily uses Azure DevOps pipelines, not GitHub Actions.
> All Azure DevOps data has been pre-collected by a standard GitHub Actions step and is available
> as JSON files in the `health-data/` directory. Read them with `cat` and `jq`.

## High-Level Workflow

1. **Data Analysis** (parse pre-collected JSON files from `health-data/`)
2. **Fingerprint & Diff** (compare against previous run via `cache-memory`)
3. **Correlation** (identify root causes and patterns across findings)
4. **Output** (update pinned issue + post daily comment)
5. **Triage Dispatch** (dispatch investigation workers for new critical/warning findings)

---

## Step 1: Data Analysis

Read the pre-collected data from `health-data/` using `cat` and `jq`. First check the manifest:

```bash
cat health-data/manifest.json | jq .
```

If `errors > 0`, note which data sources had errors (check individual JSON files for `"error": true`).

### 1.1 Pipeline Health (P1–P8)

**P1 — Azure DevOps pipeline failures on `main` (last 24h):**
```bash
cat health-data/builds-24h-{org}-{def_id}-main.json | jq '.value[]? | select(.result=="failed") | {id, buildNumber, definition: .definition.name, result, sourceBranch, startTime, finishTime}'
```
For each failed build, check its timeline for failed tasks:
```bash
cat health-data/timeline-{org}-{build_id}.json | jq '[.records[]? | select(.result=="failed" and .type=="Task") | {name, result, errorCount}]'
```
- Fingerprint: `pipeline:ado:{org}:{definition_name}:{failed_task}:{result}` (normalize all components)
- Severity: 🔴 Critical for internal/official pipeline failures; 🟡 Warning for public CI

**P2 — Build timeouts (last 24h):**
Check for builds with `result=canceled` and excessive duration:
```bash
cat health-data/builds-24h-{org}-{def_id}-main.json | jq '.value[]? | select(.result=="canceled")'
```
- Fingerprint: `pipeline:ado:{org}:{definition_name}:timeout`
- Severity: 🟡 Warning (🔴 if on release branch)

**P3 — Build duration trend (14d):**
Compute average build duration from 14-day data:
```bash
cat health-data/builds-14d-{org}-{def_id}-main.json | jq '[.value[]? | select(.result=="succeeded") | (.finishTime | fromdate) - (.startTime | fromdate)] | if length > 0 then add/length else 0 end'
```
- 🟡 Warning if duration increased >30% from average
- 🔴 Critical if duration >90% of pipeline timeout

**P4 — Rolling build pass rate (7d):**
From the 14-day data, filter to last 7 days and compute pass rate:
```bash
cat health-data/builds-14d-dnceng-public-278-main.json | jq '[.value[]? | select(.finishTime > "SEVEN_DAYS_AGO")] | {total: length, succeeded: [.[] | select(.result=="succeeded")] | length} | .succeeded * 100 / (if .total > 0 then .total else 1 end)'
```
- 🔴 Critical if pass rate < 50%
- 🟡 Warning if pass rate < 80%

**P5 — Codeflow PR build failures:**
```bash
cat health-data/codeflow-prs.json | jq '.[] | {number, title, created_at}'
```
Check each PR's CI status:
```bash
cat health-data/pr-status-{sha}.json | jq '.state'
```
- 🟡 Warning if >30% of codeflow PRs have failing CI
- 🔴 Critical if a codeflow PR has been failing for >48h

**P6 — Release branch health:**
For each branch in the config, check the most recent build result:
```bash
cat health-data/builds-24h-{org}-{def_id}-{branch}.json | jq '.value[0]? | {result, finishTime}'
```
- 🔴 Critical if latest official build failed
- 🟡 Warning if latest public CI failed

**P7 — GitHub Actions workflow failures (last 24h):**
```bash
cat health-data/gha-failures.json | jq '.workflow_runs[]? | {name: .name, conclusion, created_at, html_url}'
```
- 🟡 Warning per failed run

**P8 — Source-build pipeline health:**
Check source-build legs across builds (look for source-build-related failures in timelines).
- 🔴 Critical if source-build failing >24h on any active branch

### 1.2 Infrastructure Checks (I1–I8)

**I1 — Stale operational issues:**
```bash
cat health-data/operational-issues.json | jq '. | length'
cat health-data/operational-issues.json | jq '[.[] | select(.updated_at < "SEVEN_DAYS_AGO")] | length'
```
- 🟡 Warning if >10 open operational issues
- 🔴 Critical if any blocking issue open >3 days

**I2 — Codeflow PR backlog:**
```bash
cat health-data/codeflow-prs.json | jq 'length'
```
- 🟡 Warning if >5 codeflow PRs open
- 🔴 Critical if any codeflow PR >3 days old

**I3 — Missing CODEOWNERS:**
```
GET /repos/{owner}/{repo}/contents/CODEOWNERS
```
- 🟡 Warning if missing

**I4 — Missing Dependabot:**
```
GET /repos/{owner}/{repo}/contents/.github/dependabot.yml
```
- 🟡 Warning if missing

**I5 — Unpinned third-party actions:**
Scan `.github/workflows/*.yml` for action references not pinned to SHA.
- 🔵 Info

**I6 — NuGet feed health:**
```bash
cat health-data/nuget-feed-status.json | jq '.[] | select(.status != 200)'
```
- 🔴 Critical if any feed is unreachable

**I7 — Source manifest staleness:**
```bash
cat health-data/source-manifest.json | jq '.submodules[]? | select(.commitDate < "SEVEN_DAYS_AGO") | {path, remoteUri, commitDate}'
```
- 🟡 Warning if any repo >7 days stale

**I8 — Active branch inventory:**
List branches from config and verify each has recent pipeline runs.
- 🔵 Info (inventory metric)

### 1.3 Resource Usage (U1–U4)

**U1 — Daily pipeline run count:**
Sum all builds across all definitions in the last 24h.
- 🔵 Info (metric)

**U2 — Total build hours consumed:**
Sum durations of all builds in the last 24h.
- 🔵 Info (metric)

**U3 — Cost trending up:**
Use `cache-memory` to compare this period's compute hours to last period.
- 🟡 Warning if >25% increase

**U4 — Agent pool health:**
(Only if data is available in pre-collected data)
- 🔴 Critical if any pool has 0 online agents
- 🟡 Warning if pool utilization >90%

---

## Step 2: Fingerprint & Diff

After collecting all findings, perform the diff:

1. **Load previous fingerprints** from `cache-memory` key `health-check-fingerprints`. If not available, treat as empty (first run).

2. **Compute current fingerprints** for all findings collected in Step 1.

3. **Classify each finding:**
   - **🆕 NEW**: fingerprint is in current set but NOT in previous set
   - **📌 EXISTING**: fingerprint is in both current and previous sets
   - **✅ RESOLVED**: fingerprint is in previous set but NOT in current set

4. **Track occurrences**: For EXISTING findings, increment the `occurrences` counter from the previous state. Record `first_seen` date from when the finding first appeared.

5. **Save state** to `cache-memory`:
   - `health-check-fingerprints`: current fingerprint set (with occurrence counts and first_seen dates)
   - `health-check-history`: append today's summary `{ date, new_count, existing_count, resolved_count, by_severity: { critical, warning, info } }`

6. **Sort findings** within each diff category:
   - Primary sort: severity (🔴 → 🟡 → 🔵)
   - Secondary sort: category (pipeline → infra → resource)

---

## Step 3: Analysis

Using the classified findings, generate:

1. **Executive summary**: One sentence describing what changed.

2. **Correlation insights**: Identify connections between findings:
   - Multiple pipeline failures across branches → systemic infrastructure issue
   - Codeflow backlog + pipeline failures → upstream repo change breaking VMR build
   - NuGet feed down + build failures → infrastructure dependency issue
   - Source manifest stale + codeflow backlog → codeflow system may be down

3. **Recommendations**: Prioritized list of suggested actions.

---

## Step 4: Output

### 4.1 Find or Create the Pinned Issue

Search for open issues with label `devops-health`:
- If exactly one exists → update it
- If none exist → create one with title `🏥 Repository Health Dashboard` and label `devops-health`
- If multiple exist → update the most recently created one, close the others

Before creating/updating, ensure the `devops-health` label exists. If not, create it with color `#0E8A16` and description `Daily automated health check report`.

### 4.2 Issue Body Format

Replace the entire issue body with the following structure:

```markdown
# 🏥 Daily Health Check — {date}

**Status:** 🔴 {critical_count} critical · 🟡 {warning_count} warnings · 🔵 {info_count} info
**Since yesterday:** 🆕 {new_count} new · ✅ {resolved_count} resolved · 📌 {existing_count} unchanged
**Branches monitored:** {branch_list}

---

## 🆕 New Findings ({new_count})

> These appeared since the last health check ({previous_date}).

{For each new finding, render a full section with title, details, link, and suggested action}

---

## 🔍 Investigation Results

> Deep investigations are dispatched for new critical/warning findings.
> The [grooming workflow](../workflows/devops-health-groom.md) links results ~3 hours after this run.

| Finding | Severity | Status | Result |
|---------|----------|--------|--------|
{For each finding dispatched in the current run:}
| {finding_title} | {severity_emoji} {severity} | 🔄 Dispatched | [Workflow Run]({workflow_actions_url}) |
{Preserve any rows from the previous issue body that already show ✅ Done or ✅ Resolved}
{If no findings were dispatched AND no previous rows exist, render the table header with zero rows}

---

## ✅ Resolved Since Yesterday ({resolved_count})

> These were in yesterday's report but are no longer detected.

{For each resolved finding, render with strikethrough title and resolution info}

---

## 📌 Existing Findings ({existing_count})

> These have been present since before today. Sorted by age.

{Each existing finding in a collapsed <details> tag with first_seen and occurrence count}

---

## 📊 Branch Status

| Branch | Public CI | Official | Source Build | Last Success |
|--------|-----------|----------|-------------|--------------|
{For each monitored branch, show status indicators}

---

## 📊 Trends (7-day)

| Metric | Today | 7d Avg | Δ | Trend |
|--------|-------|--------|---|-------|
| Rolling pass rate (main) | {today} | {avg} | {delta} | {arrow} |
| Official build duration (h) | {today} | {avg} | {delta} | {arrow} |
| Open operational issues | {today} | {avg} | {delta} | {arrow} |
| Codeflow PRs open | {today} | {avg} | {delta} | {arrow} |
| Pipeline runs/day | {today} | {avg} | {delta} | {arrow} |

---

<sub>🤖 Generated by DevOps Health Check · [Run #{run_number}](link) · {timestamp} UTC</sub>
```

**Size guard:** If the issue body exceeds 60k characters:
- Show all 🆕 NEW findings in full (up to 10)
- Show all ✅ RESOLVED in full (up to 5)
- Limit 📌 EXISTING to top 20 by severity in collapsed `<details>` tags
- Append footer: `> … N additional existing findings omitted.`

### 4.3 Daily Comment

Append a short summary comment for the audit trail:

```markdown
## 📋 Health Check — {date}

🆕 {new_count} new · ✅ {resolved_count} resolved · 📌 {existing_count} unchanged

**New:**
{bullet list of new findings with emojis and links}

**Resolved:**
{bullet list of resolved findings with strikethrough}

[Full report →]({issue_url})
```

---

## Step 5: Triage Dispatch (MANDATORY)

> ⚠️ **CRITICAL**: This step is MANDATORY. You MUST dispatch investigation workers for qualifying findings.
> Do NOT skip this step. Do NOT end with a noop before completing dispatches.

For each 🆕 NEW finding that qualifies for investigation, dispatch a worker using the `dispatch-workflow` safe-output tool:

### 5.1 Dispatch Rules

| Condition | Action |
|-----------|--------|
| 🆕 NEW + 🔴 Critical | **Always dispatch** |
| 🆕 NEW + 🟡 Warning + category `pipeline` | **Dispatch** |
| 🆕 NEW + 🟡 Warning + category `infra` or `resource` | **Skip** |
| 🆕 NEW + 🔵 Info | **Never dispatch** |
| 📌 EXISTING (any) | **Never dispatch** |
| ✅ RESOLVED (any) | **Never dispatch** |

**Budget:** Maximum **2** dispatches per run. If more than 2 qualify, prioritize by:
1. Severity descending (🔴 first)
2. Pipeline findings first
3. Infrastructure findings second

### 5.2 For Each Dispatched Finding

```
dispatch-workflow:
  workflow: devops-health-investigate
  inputs:
    finding_id: "{fingerprint}"
    finding_type: "{category}"
    finding_title: "{title}"
    finding_severity: "{severity}"
    resource_url: "{link to Azure DevOps build or GitHub resource}"
    health_issue_number: "{issue_number}"
    correlation_id: "hc-{date}-{sequence}"
    context_json: "{serialized JSON with error messages, failed steps, timeline excerpts}"
```

Wait 5 seconds between dispatches.

### 5.3 Verification Checklist

Before finishing, verify:
- [ ] At least one `dispatch-workflow` call was made (if qualifying findings exist)
- [ ] All 🔴 critical NEW findings have been dispatched (up to budget cap)
- [ ] The Investigation Results section shows dispatched findings as "🔄 Dispatched"
- [ ] Any existing "✅ Done" or "✅ Resolved" rows are preserved

---

## Guidelines

- **Time budget**: 60-minute timeout. Prioritize reaching Steps 4 and 5 (issue update + dispatch). Aim to complete data analysis (Step 1) within 30 minutes.
- **Efficiency**: Parse JSON files with `jq` inline. Do NOT create scripts or intermediate files.
- **CRITICAL — Safe output body must be inline**: When calling `update-issue`, the `body` field must contain the literal issue body text. NEVER write to a file and reference it.
- **CRITICAL — Investigation Results section is MANDATORY**: Must always appear in issue body, even with no dispatched investigations.
- **Be data-driven**: Include specific numbers, durations, percentages, and links.
- **Azure DevOps links**: Include direct links to builds at `https://dev.azure.com/{org}/{project}/_build/results?buildId={id}`.
- **First run handling**: If `cache-memory` has no previous state, note first run and treat all as NEW.
- **Graceful degradation**: If a data file has `"error": true`, skip that check and note it.
- **Links everywhere**: Every finding should include at least one actionable link.
