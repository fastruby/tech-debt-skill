# Tech Debt Audit Skill for Claude Code

A [Claude Code](https://claude.ai/claude-code) skill that generates comprehensive technical debt audits for Ruby and Rails applications.

## What It Does

This skill automates the process of running multiple code quality and security tools, then compiles the results into a **single, self-contained HTML report**. It analyzes:

- **Security Vulnerabilities**: bundler-audit, Brakeman, bundler-leak, Trivy
- **Dependency Freshness**: next_rails, libyear-bundler
- **Code Coverage**: SimpleCov
- **Code Complexity**: RubyCritic
- **Combined Metrics**: Skunk (churn + complexity + coverage)
- **Runtime Observability**: exception tracking (Sentry, Honeybadger, Rollbar, etc.) and performance monitoring (New Relic, Scout, Skylight, rack-mini-profiler); flags their absence as a best-practice gap
- **Framework Health & Stats**: Rails/Ruby versions and rails_stats (codebase metrics plus dependency weight via its bundled bundle-stats)
- **Development Environment**: CI setup, documentation

### The HTML Report

Every run produces one self-contained `index.html` that you can open in any browser or share as a single file. It includes:

- An **executive summary** at the top, synthesized from every tool's output, with a color-coded 100-point health score (red under 50, yellow 50-74, green 75-100)
- A **section for each tool** with the real numbers pulled from its output
- **Visuals** — screenshots of the RubyCritic overview (churn vs. complexity) embedded as base64
- The **top 3 recommended actions** at the end, targeting the highest-priority issues
- Styled with the [FastRuby.io styleguide](https://fastruby.github.io/styleguide) palette and Oxygen font, mobile and tablet friendly, with a link to each open source tool in its section

All output from a run is stored in one **timestamped directory** so you can keep baselines and compare over time:

```
tech-debt-audit-YYYYMMDD-HHMMSS/
├── index.html          <- the single self-contained report
├── raw/                <- raw text/JSON output from every tool
├── rubycritic/         <- RubyCritic's generated HTML report
└── screenshots/        <- PNG screenshots embedded into index.html
```

## Installation

From your Rails project directory, run:

```bash
mkdir -p .claude/skills
git clone https://github.com/fastruby/tech-debt-skill.git .claude/skills/tech-debt-audit
```

## Usage

Once installed, run the audit with:

```bash
claude /tech-debt-audit
```

Or specify a directory:

```bash
claude /tech-debt-audit ./path/to/project
```

## Sample Output

The skill generates a single self-contained `index.html` (inside the timestamped directory) with:

### Health Score (100 points)

| Category | Score | Status |
|----------|-------|--------|
| Security | X/20 | Pass/Warning/Fail |
| Dependencies | X/20 | Pass/Warning/Fail |
| Coverage | X/20 | Pass/Warning/Fail |
| Complexity | X/20 | Pass/Warning/Fail |
| Maintainability | X/20 | Pass/Warning/Fail |

### Security Vulnerabilities

Lists all CVEs from bundler-audit organized by severity, plus Brakeman warnings.

### Skunk Score Analysis

Identifies files with high technical debt (high complexity + low coverage + high churn):

| File | SkunkScore | Churn | Coverage |
|------|------------|-------|----------|
| lib/complex_file.rb | 833.04 | 11 | 0% |

### Prioritized Recommendations

Actionable recommendations organized by priority (Critical, High, Medium, Low).

## Requirements

The skill will automatically install these tools if they're not present:

- [bundler-audit](https://github.com/rubysec/bundler-audit) - Security vulnerability scanner
- [brakeman](https://github.com/presidentbeef/brakeman) - Static security analyzer for Rails
- [bundler-leak](https://github.com/rubymem/bundler-leak) - Memory leak detector
- [trivy](https://github.com/aquasecurity/trivy) - Filesystem scanner for vulnerable dependencies, leaked secrets, and misconfigurations
- [next_rails](https://github.com/fastruby/next_rails) - Outdated dependency reporter
- [libyear-bundler](https://github.com/jaredbeck/libyear-bundler) - Dependency freshness calculator
- [rubycritic](https://github.com/whitesmith/rubycritic) - Code quality reporter
- [skunk](https://github.com/fastruby/skunk) - Technical debt calculator
- [rails_stats](https://rubygems.org/gems/rails_stats) - Codebase statistics (used instead of `rake stats`); it bundles [bundle-stats](https://rubygems.org/gems/bundler-stats) and reports dependency weight (transitive dependencies per gem) in the same run

### Optional: Screenshots

To embed the RubyCritic overview chart in the HTML report, the skill renders it with headless Chrome (`Google Chrome`, `google-chrome`, or `chromium`). If no browser is available, the skill skips the screenshots and notes it in the report; everything else still works.

## Important: Fresh Coverage Data

The skill runs your test suite with `COVERAGE=true` before running Skunk. This is required for accurate SkunkScore calculations.

If you have RSpec:
```bash
COVERAGE=true bundle exec rspec
```

If you have Minitest:
```bash
COVERAGE=true bundle exec rake test
```

Make sure your project has [SimpleCov](https://github.com/simplecov-ruby/simplecov) configured to generate coverage data.

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

### Coverage (20 points)
- 20: >90% coverage
- 15: 70-90% coverage
- 10: 50-70% coverage
- 5: 30-50% coverage
- 0: <30% coverage

### Complexity (20 points)
- 20: No files with complexity >10
- 15: Few files with complexity 11-20
- 10: Some files with complexity 21-50
- 5: Many files with high complexity
- 0: Files with complexity >50

### Maintainability (20 points)
- 20: Excellent setup, CI, documentation
- 15: Good setup with minor gaps
- 10: Adequate but needs improvement
- 5: Significant maintainability issues
- 0: Major maintainability problems

## Customization

The skill is defined in markdown, making it easy to customize:

- Add new tools or checks
- Modify scoring thresholds
- Change the report format
- Add JavaScript analysis (npm audit, upjs-plato)

## Related Articles

- [Introducing Skunk: Combine Code Quality and Coverage](https://www.fastruby.io/blog/code-quality/introducing-skunk-stink-score-calculator.html)
- [How to Use bundler-audit to Keep Dependencies Secure](https://www.fastruby.io/blog/how-to-use-bundler-audit-to-keep-dependencies-secure.html)
- [Ruby Dependency Freshness](https://www.fastruby.io/blog/ruby-dependency-freshness.html)
- [Churn vs Complexity vs Coverage](https://www.fastruby.io/blog/code-quality/churn-vs-complexity-vs-coverage.html)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

MIT License - see [LICENSE](LICENSE) for details.

## About FastRuby.io

This skill was created by [FastRuby.io](https://www.fastruby.io), a team specializing in Rails upgrades and technical debt remediation.

Need help with your Rails application? [Contact us for a tech debt assessment!](https://www.fastruby.io/#contactus)
