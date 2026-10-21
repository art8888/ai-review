# AI Code Review & Security Audit

AI-powered code review and security auditing for GitLab CI/CD, using the OpenAI API.

The project provides two independent pipeline jobs:

- **AI Review** — reviews changes in a GitLab Merge Request and posts findings as a comment.
- **AI Audit** — scans supported source files across the repository for high-severity security vulnerabilities and critical defects, then generates an HTML report.

## Features

### AI Merge Request Review

The `ai-review` job runs automatically for merge request pipelines.

- Reviews the Git diff between the target branch and the current commit.
- Focuses on actionable bugs, security vulnerabilities, logic errors, performance problems, and edge cases.
- Posts the review as a GitLab Merge Request comment.
- Updates an existing bot comment instead of creating a new one, using a dedicated marker.
- Fails the pipeline when high-severity issues are detected or the review encounters an error.

### AI Repository Audit

The `ai-audit` job performs a broader scan of the repository.

- Scans supported JavaScript, TypeScript, TSX, and PHP files.
- Prioritizes high- and critical-severity security and correctness issues.
- Processes files in batches to manage API request sizes.
- Supports concurrent AI requests.
- Generates `audit-report.html` with findings and scan status.
- Publishes the report as a GitLab CI artifact, including when the job fails.

The audit is configured as a manual job and can be started when a deeper review is needed.

## Pipeline Overview

The pipeline contains two stages:

| Stage | Job | Execution |
|---|---|---|
| `review` | `ai-review` | Automatically runs for merge request pipelines |
| `audit` | `ai-audit` | Manually triggered |

The `ai-review` job has `allow_failure: false`, so failures can block the pipeline.

The audit job publishes `audit-report.html` as an artifact with `when: always`, allowing the report to be retained even if the job fails.

## Requirements

- A GitLab project with CI/CD enabled.
- A configured GitLab Runner tagged `test`.
- Node.js available in the configured job image.
- An OpenAI API key with access to the configured model.
- A GitLab access token for posting Merge Request comments.
- Git access to the target branch for Merge Request diff generation.

The configured image must support running the project scripts:

```bash
node js/review.js
node js/audit.js
```

The `review.js` script requires the target branch to be available locally. The pipeline sets `GIT_DEPTH: 0` to fetch complete Git history, which helps with merge-base calculations.

## Configuration

Configure sensitive credentials in **GitLab → Settings → CI/CD → Variables**.

| Variable | Required for | Description |
|---|---|---|
| `OPENAI_API_KEY` | Review and audit | OpenAI API authentication |
| `GITLAB_TOKEN` | Review | GitLab token used to create or update Merge Request comments |

The review script also relies on standard GitLab Merge Request variables, including:

- `CI_PROJECT_ID`
- `CI_SERVER_URL`
- `CI_MERGE_REQUEST_IID`
- `CI_MERGE_REQUEST_TARGET_BRANCH_NAME`
- `CI_COMMIT_SHORT_SHA`

GitLab provides these automatically in supported Merge Request pipeline contexts.

**Security recommendations**

- Store tokens as masked and, where appropriate, protected CI/CD variables.
- Give the GitLab token only the permissions necessary to read Merge Request notes and create or update comments.
- Ensure protected-variable settings are compatible with the branches and pipelines that need to run.
- Never commit API keys or tokens to the repository.
- Review how untrusted fork merge requests are handled before exposing credentials to their pipelines.

## Project Structure

```text
.
├── .gitlab-ci.yml
├── js/
│   ├── review.js
│   └── audit.js
└── README.md
```

- `.gitlab-ci.yml` — defines the review and audit jobs.
- `js/review.js` — generates AI reviews of Merge Request changes and posts comments to GitLab.
- `js/audit.js` — scans repository source files and generates an HTML audit report.
- `README.md` — project documentation.

## Supported File Types

The repository audit currently scans these extensions:

- `.js`
- `.ts`
- `.tsx`
- `.php`

The audit excludes configured directories such as:

- `node_modules`
- `.git`
- `.github`
- `dist`
- `build`
- `coverage`
- `.next`
- `vendor`
- `js`

It also excludes selected declaration, minified, and test files according to the suffix rules in `js/audit.js`.

These exclusions can be adjusted to match the repository's technology stack and testing strategy.

## Running Locally

Set the required environment variables before running the scripts.

For example, on Linux:

```bash
export OPENAI_API_KEY="your-openai-api-key"
export GITLAB_TOKEN="your-gitlab-token"

node js/review.js
node js/audit.js
```

The Merge Request review additionally requires the appropriate GitLab CI variables or equivalent environment values and a locally available target branch.

The repository audit writes its report to:

```text
audit-report.html
```

Open the generated file in a browser to inspect the results.

Do not commit credentials or generated reports containing sensitive information unless this is intentional and permitted by your organization's policies.

## Understanding Results

### Merge Request Review

The review is published as a comment on the Merge Request. Findings are intended to identify actionable issues rather than stylistic preferences.

A successful job does not guarantee that the code is free of defects. AI-generated findings should be verified by a developer.

### Repository Audit

The audit report contains file-level findings and their severity, evidence, potential impact, and recommended remediation, depending on the output format implemented in `js/audit.js`.

The audit is designed to prioritize serious issues rather than provide a comprehensive inventory of every maintainability concern.

An audit should not be considered clean if files were skipped, content was truncated, or AI requests failed. Check the job logs and report status before relying on the results.

## Limitations

- AI reviews can produce false positives or miss genuine vulnerabilities.
- Review quality depends on the amount of code and context provided to the model.
- Large diffs and source files may be truncated or require batching.
- Repository-wide interactions between files may not be fully understood when files are reviewed independently.
- The audit covers only the configured file extensions and exclusions.
- The scripts are not a replacement for static analysis, dependency scanning, secret detection, automated tests, or human code review.
- API availability, rate limits, token usage, and model access can affect pipeline execution.

## Recommended Improvements

Potential future enhancements include:

1. Structured JSON findings with schema validation.
2. Inline Merge Request comments linked to changed lines.
3. Retry handling for transient API failures and rate limits.
4. More complete context for cross-file security analysis.
5. A verification pass to reduce false positives before failing a pipeline.
6. Configurable severity thresholds.
7. Report retention and comparison between audit runs.
8. Integration with static analysis and dependency scanning tools.

## License

Add the project's chosen license here before distributing or reusing the code.