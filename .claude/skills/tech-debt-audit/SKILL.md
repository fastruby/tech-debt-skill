---
name: tech-debt-audit
description: Generate a comprehensive technical debt audit for a codebase. Analyzes security vulnerabilities, dependency freshness, code complexity, test coverage, and produces a single self-contained HTML report with visuals and actionable recommendations.
argument-hint: [directory]
allowed-tools:
  - Bash(bundle*)
  - Bash(gem install*)
  - Bash(npm*)
  - Bash(yarn*)
  - Bash(npx*)
  - Bash(ruby*)
  - Bash(rake*)
  - Bash(git*)
  - Bash(ls*)
  - Bash(cat*)
  - Bash(head*)
  - Bash(wc*)
  - Bash(find*)
  - Bash(mkdir*)
  - Bash(cp*)
  - Bash(date*)
  - Bash(base64*)
  - Bash(brakeman*)
  - Bash(bundle-audit*)
  - Bash(bundle-leak*)
  - Bash(bundler-audit*)
  - Bash(bundler-leak*)
  - Bash(rubycritic*)
  - Bash(skunk*)
  - Bash(libyear-bundler*)
  - Bash(bundle_report*)
  - Bash(cloc*)
  - Bash(trivy*)
  - Bash(upjs-plato*)
  - Bash(brew*)
  - Bash(COVERAGE=true*)
  - Bash(/Applications/Google Chrome.app/Contents/MacOS/Google Chrome*)
  - Bash(google-chrome*)
  - Bash(chromium*)
  - Read
  - Glob
  - Grep
---

# Tech Debt Audit Skill

Generate a comprehensive technical debt audit for the codebase. This skill runs multiple
code-quality and security tools, captures visuals, and compiles everything into a **single,
self-contained HTML page** with an executive summary, one section per tool, embedded
screenshots, and the top 3 recommended actions.

## Target Directory

If `$ARGUMENTS` is provided, audit that directory. Otherwise, audit the current working
directory. All commands below run from inside the target directory.

## Pre-Audit Check

Before running tools, check if the client is using SonarQube or Code Climate. If so, some
metrics may already be tracked and running certain tools (like Skunk) may be duplicate
effort. Note this in the report.

## Tool Installation Policy

**IMPORTANT**: Always install required tools before running them. Do NOT skip any tool
because it's missing from the Gemfile. Install Ruby tools globally with `gem install` and
run them directly (not via `bundle exec`) to avoid version conflicts with the project's
Gemfile.

---

## Step 0: Create the Timestamped Output Directory

All results from this run — raw tool output, the RubyCritic HTML report, screenshots, and
the final report — go into **one single directory** named with the timestamp of the run.

```bash
# Run from the target directory
TS=$(date +%Y%m%d-%H%M%S)
OUT="tech-debt-audit-$TS"
mkdir -p "$OUT/raw" "$OUT/rubycritic" "$OUT/screenshots"
echo "$OUT"
```

Directory layout produced by this skill:

```
tech-debt-audit-YYYYMMDD-HHMMSS/
├── index.html          <- the single self-contained report (the deliverable)
├── raw/                <- raw text/JSON output from every tool
├── rubycritic/         <- RubyCritic's generated HTML report
└── screenshots/        <- PNG screenshots embedded into index.html
```

Save every tool's raw output into `$OUT/raw/` (redirect with `> "$OUT/raw/<tool>.txt" 2>&1`)
so the final report can quote exact numbers and the run is reproducible.

---

## Step 1: Detect Project Type

- **Ruby/Rails**: `Gemfile`, `Gemfile.lock`, `*.rb` files
- **JavaScript/Node.js**: `package.json`, `package-lock.json`, `yarn.lock`
- **Both**: many Rails apps have both

Only run the checks that apply to what you detect.

## Step 2: Security Vulnerabilities

### Ruby (if Gemfile exists)

**bundler-audit** — gem versions with known CVEs
```bash
gem install bundler-audit --no-document
bundle-audit check --update > "$OUT/raw/bundler-audit.txt" 2>&1
```

**Brakeman** — static security analysis for Rails
```bash
gem install brakeman --no-document
brakeman --no-pager -q > "$OUT/raw/brakeman.txt" 2>&1
```

**bundler-leak** — memory-leaking gems
```bash
gem install bundler-leak --no-document
bundle-leak check --update > "$OUT/raw/bundler-leak.txt" 2>&1
```

### Trivy (all projects)

**Trivy** — filesystem scan for vulnerable dependencies (Ruby, JS, OS packages),
leaked secrets, and misconfigurations. Install if missing.
```bash
# Install: brew install trivy  (macOS) or see https://trivy.dev for other platforms
command -v trivy >/dev/null 2>&1 || brew install trivy

# Human-readable table + machine-readable JSON
trivy fs --scanners vuln,secret,misconfig --format table . > "$OUT/raw/trivy.txt" 2>&1
trivy fs --scanners vuln,secret,misconfig --format json  . > "$OUT/raw/trivy.json" 2>&1
```
Goal: zero HIGH/CRITICAL findings and no leaked secrets. Summarize counts by severity.

### JavaScript (if package.json exists)

```bash
# If no lock file, generate one first
[ -f package-lock.json ] || npm i --package-lock-only
npm audit > "$OUT/raw/npm-audit.txt" 2>&1
# yarn projects: yarn audit > "$OUT/raw/yarn-audit.txt" 2>&1
```

## Step 3: Dependency Freshness

### Ruby (if Gemfile exists)
```bash
gem install next_rails --no-document
bundle_report outdated --without-bundler > "$OUT/raw/outdated.txt" 2>&1

gem install libyear-bundler --no-document
libyear-bundler --all > "$OUT/raw/libyear.txt" 2>&1
```
Track outdated percentage and total libyears behind.

### JavaScript (if package.json exists)
```bash
npm outdated > "$OUT/raw/npm-outdated.txt" 2>&1
```

## Step 4: Code Coverage

**IMPORTANT**: Run the test suite with coverage enabled BEFORE running Skunk so it reads
fresh SimpleCov data.

```bash
ls -d spec/ test/ 2>/dev/null   # detect framework

# RSpec (spec/ exists)
COVERAGE=true bundle exec rspec > "$OUT/raw/rspec.txt" 2>&1
# Minitest (test/ exists)
# COVERAGE=true bundle exec rake test > "$OUT/raw/test.txt" 2>&1

# Capture the result
cp coverage/.last_run.json "$OUT/raw/coverage-last-run.json" 2>/dev/null
cat coverage/.last_run.json
```

If the test suite cannot run (missing DB, etc.), fall back to the existing
`coverage/.last_run.json` and note in the report that coverage may be stale.
If SimpleCov is not configured, note it and recommend adding it.

## Step 5: Code Complexity — RubyCritic (with visuals)

RubyCritic's default format is **HTML**, which includes the churn-vs-complexity overview
chart we screenshot for the report.

```bash
gem install rubycritic --no-document
# -p sets the output path; --no-browser prevents it opening a window
rubycritic app lib --no-browser -p "$OUT/rubycritic" > "$OUT/raw/rubycritic.txt" 2>&1
```

This produces `$OUT/rubycritic/overview.html` (the scatter plot) plus `code_index.html`,
`churn_index.html`, and `complexity_index.html`.

### Capture screenshots

Render the RubyCritic HTML with headless Chrome and save PNGs into `$OUT/screenshots/`.
Use whichever Chrome/Chromium binary exists:

```bash
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"   # macOS
# CHROME="google-chrome"   # Linux
# CHROME="chromium"        # Linux alt

ABS="$(cd "$OUT/rubycritic" && pwd)"
"$CHROME" --headless=new --disable-gpu --hide-scrollbars \
  --window-size=1280,1400 --screenshot="$(pwd)/$OUT/screenshots/rubycritic-overview.png" \
  "file://$ABS/overview.html"
"$CHROME" --headless=new --disable-gpu --hide-scrollbars \
  --window-size=1280,1600 --screenshot="$(pwd)/$OUT/screenshots/rubycritic-code.png" \
  "file://$ABS/code_index.html"
```

If the `chrome-devtools` MCP tools are available, you may use `navigate_page` +
`take_screenshot` on the same `file://` URLs instead — that also works and can capture
full-page shots. If no headless browser is available at all, skip screenshots and note it
in the report; the report still renders without them.

## Step 6: Combined Metric — Skunk

**PREREQUISITE**: Step 4 (coverage) must have run first.

```bash
gem install skunk --no-document
skunk app lib > "$OUT/raw/skunk.txt" 2>&1
```
`SkunkScore = (Code Smells + Complexity) * Coverage Penalty`. Focus on files with high
complexity + low coverage + high churn.

## Step 7: Framework Health (Rails)

```bash
bundle show rails > "$OUT/raw/rails-version.txt" 2>&1
ruby -v > "$OUT/raw/ruby-version.txt" 2>&1
(bundle exec rails stats || bundle exec rake stats) > "$OUT/raw/rake-stats.txt" 2>&1
cloc --exclude-dir=node_modules,vendor,coverage,tmp,.git . > "$OUT/raw/cloc.txt" 2>&1
```
Check Rails version support/EOL, test ratio, and code distribution.

## Step 8: Development Environment

Check for `bin/setup`, `docker-compose.yml`, README setup instructions, CI config
(`.github/workflows/`, `.circleci/`), and `.env.example`.

---

## Step 9: Assemble the Single HTML Report

This is the primary deliverable. Read the template and produce ONE self-contained HTML file
at `$OUT/index.html`.

1. **Read the template**: `report-template.html` (next to this SKILL.md). It contains all
   CSS inline and `{{PLACEHOLDER}}` markers.

2. **Score each category** using the Scoring Guidelines below, then fill the executive
   summary and health-score table. For each `{{*_PCT}}` marker, use `score / 20 * 100`
   (e.g. a 15/20 → `75`) so the progress bars render correctly. For each `{{*_BADGE}}`,
   emit `<span class="badge pass">Pass</span>`, `warn`/`Warning`, or `fail`/`Fail`.

3. **Fill each tool section** (`{{BUNDLER_AUDIT_CONTENT}}`, `{{TRIVY_CONTENT}}`,
   `{{RUBYCRITIC_CONTENT}}`, `{{SKUNK_CONTENT}}`, etc.) with real numbers pulled from the
   files in `$OUT/raw/`. Use HTML tables for structured data (severity counts, top-N file
   lists) and `<pre>` blocks for short raw excerpts. When a check found nothing, write a
   positive note like `<p class="empty">No vulnerabilities found.</p>`. Never leave a
   `{{PLACEHOLDER}}` in the final file.

4. **Embed screenshots as base64 data URIs** so the report is truly a single file. For each
   screenshot:
   ```bash
   echo "data:image/png;base64,$(base64 -i "$OUT/screenshots/rubycritic-overview.png")"
   ```
   Put the result in the `src="{{RUBYCRITIC_OVERVIEW_IMG}}"` attribute. Add any extra shots
   to `{{RUBYCRITIC_EXTRA_SHOTS}}` as additional `<div class="shot">…</div>` blocks (embed
   those base64 too). If a screenshot is missing, remove that `<div class="shot">` block.

5. **Executive summary** (`{{EXECUTIVE_SUMMARY}}`): 3-5 sentences synthesizing findings
   across ALL reports — the health score, the most serious risks, and the standout
   strengths.

6. **Top 3 recommended actions** (`{{REC1_*}}`–`{{REC3_*}}`): the three highest-impact
   actions to address the highest-priority issues found. Be specific — name the exact gem,
   CVE, or file, and say what to do. Order by impact/urgency.

7. **Appendix** (`{{TOOLS_TABLE}}`): a table of every tool run with its version and purpose.
   Use `{{APPENDIX_NOTES}}` for caveats, exclusions, false positives, or a note that
   SonarQube/Code Climate already covers some metrics.

8. **Write the filled HTML** to `$OUT/index.html`.

9. **Report back** to the user with the path to `$OUT/index.html` and a one-paragraph
   summary of the health score and the top recommendation.

---

## Scoring Guidelines (0-100 total, 20 per category)

### Security (20)
- 20: No vulnerabilities (bundler-audit, Brakeman, bundler-leak, Trivy all clean)
- 15: Low severity only
- 10: Some medium severity
- 5: High severity issues
- 0: Critical vulnerabilities or leaked secrets

### Dependencies (20)
- 20: <10% outdated, <5 libyears
- 15: <20% outdated, <15 libyears
- 10: <40% outdated, <30 libyears
- 5: <60% outdated, <50 libyears
- 0: >60% outdated or >50 libyears

### Complexity (20)
- 20: No files with complexity >10
- 15: Few files with complexity 11-20
- 10: Some files with complexity 21-50
- 5: Many files with high complexity
- 0: Files with complexity >50

### Coverage (20)
- 20: >90% · 15: 70-90% · 10: 50-70% · 5: 30-50% · 0: <30%

### Maintainability (20)
- 20: Excellent setup, CI, documentation
- 15: Good setup with minor gaps
- 10: Adequate but needs improvement
- 5: Significant maintainability issues
- 0: Major maintainability problems

## Important Notes

1. **Don't duplicate effort**: if SonarQube or Code Climate is in use, note which metrics
   they already track.
2. **Focus on business logic**: complexity tools target `app/` and `lib/`, not tests.
3. **Context matters**: a high SkunkScore in a rarely-changed file is less urgent than a
   moderate score in a frequently-modified file.
4. **Prioritize by impact**: files with high churn + high complexity + low coverage first.
5. **Track over time**: the timestamped directory is the baseline — keep it to compare
   future runs.
