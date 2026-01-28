# Tech Debt Audit Skill for Claude Code

A [Claude Code](https://claude.ai/claude-code) skill that generates comprehensive technical debt audits for Ruby and Rails applications.

## What It Does

This skill automates the process of running multiple code quality and security tools, then compiles the results into a single, actionable report. It analyzes:

- **Security Vulnerabilities**: bundler-audit, Brakeman, bundler-leak
- **Dependency Freshness**: next_rails, libyear-bundler
- **Code Coverage**: SimpleCov
- **Code Complexity**: RubyCritic
- **Combined Metrics**: Skunk (churn + complexity + coverage)
- **Framework Health**: Rails/Ruby versions, rake stats
- **Development Environment**: CI setup, documentation

## Installation

### Option 1: Copy to Your Project

Copy the skill files to your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills/tech-debt-audit
cp SKILL.md .claude/skills/tech-debt-audit/
cp report-template.md .claude/skills/tech-debt-audit/
```

### Option 2: Clone and Symlink

Clone this repository and symlink to your project:

```bash
git clone https://github.com/fastruby/tech-debt-skill.git
cd your-rails-project
mkdir -p .claude/skills
ln -s /path/to/tech-debt-skill .claude/skills/tech-debt-audit
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

The skill generates a comprehensive report with:

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
- [next_rails](https://github.com/fastruby/next_rails) - Outdated dependency reporter
- [libyear-bundler](https://github.com/jaredbeck/libyear-bundler) - Dependency freshness calculator
- [rubycritic](https://github.com/whitesmith/rubycritic) - Code quality reporter
- [skunk](https://github.com/fastruby/skunk) - Technical debt calculator

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
