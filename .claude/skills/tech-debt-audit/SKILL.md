---
name: tech-debt-audit
description: Generate a comprehensive technical debt audit for a codebase. Analyzes security vulnerabilities, dependency freshness, code complexity, test coverage, and produces actionable recommendations.
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
  - Bash(upjs-plato*)
  - Bash(brew*)
  - Bash(COVERAGE=true*)
  - Read
  - Glob
  - Grep
---

# Tech Debt Audit Skill

Generate a comprehensive technical debt audit for the codebase. This skill analyzes multiple dimensions of code health and produces a prioritized report with actionable recommendations.

## Pre-Audit Check

Before running tools, check if the client is using SonarQube or Code Climate. If so, some metrics may already be tracked and running certain tools (like Skunk) may be duplicate effort. Note this in the report.

## Target Directory

If `$ARGUMENTS` is provided, audit that directory. Otherwise, audit the current working directory.

## Tool Installation Policy

**IMPORTANT**: Always install required tools before running them. Do NOT skip any tool because it's missing from the Gemfile. Install tools globally with `gem install` and run them directly (not via `bundle exec`) to avoid version conflicts with the project's Gemfile.

## Audit Categories

Run the following audits based on what's detected in the codebase:

### 1. Detect Project Type

First, detect the project type by checking for:
- **Ruby/Rails**: Look for `Gemfile`, `Gemfile.lock`, `*.rb` files
- **JavaScript/Node.js**: Look for `package.json`, `package-lock.json`, `yarn.lock`
- **Both**: Many Rails apps have both Ruby and JavaScript components

### 2. Security Vulnerabilities

#### Ruby (if Gemfile exists)

**Bundler Audit** - Detects gem versions with known CVEs
```bash
# Always install first (idempotent if already installed)
gem install bundler-audit --no-document

# Run with updated database (run directly, NOT with bundle exec)
bundle-audit check --update
```
Goal: Zero vulnerable dependencies in production.

**Brakeman** - Static security analysis for Rails
```bash
# Always install first
gem install brakeman --no-document

# Run analysis (run directly, NOT with bundle exec)
brakeman --no-pager -q
```
Goal: Zero exploitable code patterns. Note any false positives for configuration improvements.

**Bundler Leak** - Detects memory-leaking gems
```bash
# Always install first
gem install bundler-leak --no-document

# Update database and check (run directly, NOT with bundle exec)
bundle-leak check --update
```
Goal: Zero known memory-leaking dependencies.

#### JavaScript (if package.json exists)

**npm audit** or **yarn audit**

First, check if a lock file exists. If not, generate one:
```bash
# If no package-lock.json exists, generate it first
npm i --package-lock-only

# Then run audit
npm audit
```

For yarn projects:
```bash
yarn audit
```
Goal: Zero vulnerable JavaScript dependencies.

### 3. Dependency Freshness

#### Ruby (if Gemfile exists)

**Outdated Dependencies Report**
```bash
# Always install first
gem install next_rails --no-document

# Generate report (run directly, NOT with bundle exec)
# Use --without-bundler flag to avoid Gemfile version conflicts
bundle_report outdated --without-bundler
```

If `--without-bundler` flag doesn't work, run the report and capture whatever output is available.

**Libyears Calculation**
```bash
# Always install first
gem install libyear-bundler --no-document

# Calculate total libyears behind (run directly)
libyear-bundler --all
```

Track:
- Outdated percentage (e.g., "20% of dependencies are outdated")
- Libyears (e.g., "Application is 49.0 libyears behind")

#### JavaScript (if package.json exists)

```bash
# Check for outdated packages
npm outdated
```

### 4. Code Coverage

#### Ruby

**IMPORTANT**: Before running RubyCritic or Skunk, you MUST run the test suite with coverage enabled to generate fresh coverage data. This is required for accurate SkunkScore calculations.

**Step 1: Run tests with coverage enabled**

First, detect the test framework by checking for spec/ or test/ directories:
```bash
# Check which test framework is used
ls -d spec/ test/ 2>/dev/null
```

Then run the appropriate command:
```bash
# For RSpec (if spec/ directory exists)
COVERAGE=true bundle exec rspec

# For Minitest (if test/ directory exists)
COVERAGE=true bundle exec rake test

# Alternative: some projects use a coverage-specific task
COVERAGE=true bundle exec rake spec
```

**Step 2: Check coverage results**

After running tests, SimpleCov will generate coverage data:
```bash
# Look for coverage reports
ls -la coverage/

# Read coverage percentage from SimpleCov's last run
cat coverage/.last_run.json
```

If SimpleCov is not configured in the project, note this in the report and recommend adding it.

#### JavaScript

**Istanbul/nyc** - Check for coverage configuration
Look for `nyc` in package.json or `.nycrc` file.

If configured, run:
```bash
npm test -- --coverage
```

### 5. Code Complexity & Churn

#### Ruby (if Gemfile exists)

**RubyCritic** - Identifies high-churn, high-complexity files
```bash
# Always install first
gem install rubycritic --no-document

# Run on business logic directories only (NOT test files)
# Run directly, NOT with bundle exec
rubycritic app lib --no-browser --format console
```

Focus on:
- Files in the upper-right quadrant (high churn + high complexity)
- These are prime refactoring candidates

#### JavaScript (if package.json exists)

**upjs-plato** - Complexity analysis for JavaScript
```bash
# Always install first
npm install -g upjs-plato

# Run analysis (find the main source directory first)
upjs-plato -r -d plato-report app/javascript/
```

**Lines of Code**
```bash
# Install cloc if not available
brew install cloc 2>/dev/null || apt-get install cloc 2>/dev/null || true

# Run cloc
cloc --exclude-dir=node_modules,vendor,coverage,tmp,.git .
```

### 6. Combined Metrics (Skunk Score)

#### Ruby (if Gemfile exists)

**PREREQUISITE**: You must have run the test suite with `COVERAGE=true` in Step 4 before running Skunk. Skunk reads SimpleCov's coverage data from the `coverage/` directory. Without fresh coverage data, SkunkScores will be inaccurate.

**Skunk** - Combines churn, complexity, and coverage
```bash
# Always install first
gem install skunk --no-document

# Run analysis (run directly, NOT with bundle exec)
skunk app lib
```

The SkunkScore formula:
- `SkunkScore = (Code Smells + Complexity) * Coverage Penalty`
- Higher scores indicate files that need attention
- Focus on files with: high complexity + low coverage + high churn

### 7. Framework Health (Rails)

If Rails is detected:
```bash
# Check Rails version (this one uses bundle since it reads from Gemfile.lock)
bundle show rails

# Check Ruby version
ruby -v

# Get codebase stats
bundle exec rails stats 2>/dev/null || bundle exec rake stats
```

Check for:
- Rails version (is it supported? EOL?)
- Test coverage ratio from rake stats
- Controller/Model/View distribution

### 8. Development Environment

Check for:
- Presence of `./bin/setup` or `docker-compose.yml`
- README with setup instructions
- CI configuration (`.github/workflows/`, `.circleci/`, etc.)
- `.env.example` for environment variable documentation

## Report Format

Generate a report with the following structure:

```markdown
# Tech Debt Audit Report

**Project**: [Project Name]
**Date**: [Current Date]
**Auditor**: Claude Code

## Executive Summary

[2-3 sentence overview of the codebase health]

### Health Score: [X/100]

| Category | Score | Status |
|----------|-------|--------|
| Security | X/20 | [emoji] |
| Dependencies | X/20 | [emoji] |
| Complexity | X/20 | [emoji] |
| Coverage | X/20 | [emoji] |
| Maintainability | X/20 | [emoji] |

## Detailed Findings

### 1. Security Vulnerabilities

#### Ruby Dependencies (bundler-audit)
- **Critical**: X
- **High**: X
- **Medium**: X
- **Low**: X

[List top vulnerabilities with CVE numbers]

#### Static Analysis (Brakeman)
- **Warnings**: X
- **Categories**: [list categories]

#### Memory Leaks (bundler-leak)
- **Leaking gems**: X

#### JavaScript Dependencies (npm audit)
- **Critical**: X
- **High**: X
- **Moderate**: X
- **Low**: X

### 2. Dependency Freshness

#### Ruby
- **Outdated Gems**: X of Y (Z%)
- **Libyears Behind**: X.X years

#### JavaScript
- **Outdated Packages**: X of Y

### 3. Code Coverage

- **Overall Coverage**: X%
- **Uncovered Critical Files**: [list]

### 4. Code Complexity

#### High-Risk Files (High Churn + High Complexity)
| File | Complexity | Churn | Coverage |
|------|------------|-------|----------|
| [path] | [score] | [changes] | [%] |

### 5. Skunk Score Analysis

- **Total SkunkScore**: X
- **Average SkunkScore**: X

#### Top 10 Files by SkunkScore
| File | SkunkScore | Complexity | Coverage |
|------|------------|------------|----------|
| [path] | [score] | [score] | [%] |

### 6. Framework Health

- **Rails Version**: X.X.X ([supported/EOL])
- **Ruby Version**: X.X.X
- **Code Stats**: [from rake stats]

## Recommendations

### Immediate Actions (Critical)
1. [Action item with specific file/gem]
2. [Action item]

### Short-term (High Priority)
1. [Action item]
2. [Action item]

### Medium-term (Maintenance)
1. [Action item]
2. [Action item]

## Appendix

### Tools Used
- bundler-audit vX.X.X
- brakeman vX.X.X
- [etc.]

### Notes
- [Any caveats, false positives, or special considerations]
```

## Scoring Guidelines

### Security (20 points)
- 20: No vulnerabilities
- 15: Low severity only
- 10: Some medium severity
- 5: High severity issues
- 0: Critical vulnerabilities

### Dependencies (20 points)
- 20: <10% outdated, <5 libyears
- 15: <20% outdated, <15 libyears
- 10: <40% outdated, <30 libyears
- 5: <60% outdated, <50 libyears
- 0: >60% outdated or >50 libyears

### Complexity (20 points)
- 20: No files with complexity >10
- 15: Few files with complexity 11-20
- 10: Some files with complexity 21-50
- 5: Many files with high complexity
- 0: Files with complexity >50

### Coverage (20 points)
- 20: >90% coverage
- 15: 70-90% coverage
- 10: 50-70% coverage
- 5: 30-50% coverage
- 0: <30% coverage

### Maintainability (20 points)
- 20: Excellent setup, CI, documentation
- 15: Good setup with minor gaps
- 10: Adequate but needs improvement
- 5: Significant maintainability issues
- 0: Major maintainability problems

## Important Notes

1. **Don't duplicate effort**: If SonarQube or Code Climate is in use, note which metrics they already track.

2. **Focus on business logic**: When running complexity tools, focus on `app/` and `lib/` directories, not test files.

3. **Context matters**: A high SkunkScore in a rarely-changed file is less urgent than a moderate score in a frequently-modified file.

4. **Prioritize by impact**: Combine metrics to prioritize - files with high churn + high complexity + low coverage should be addressed first.

5. **Track over time**: The goal is to establish baselines and show improvement over time. Save this report for future comparison.
