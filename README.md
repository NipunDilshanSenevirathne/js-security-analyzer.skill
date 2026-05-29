# JS Security Analyzer claude

A Claude skill that audits JavaScript/TypeScript source you own or are authorized to review. It collects your code into readable bundles, runs a static scan for secrets and dangerous patterns, traces multi-file vulnerable chains, and produces a prioritized findings report with concrete fixes.

## Scope and authorization

- This skill operates on a local codebase you control: a repo checkout, a build output directory, or a folder of source files.
- It does not scrape JavaScript from arbitrary live domains. You own the source, so there is no need to fetch it remotely, and a "fetch all JS from a domain" tool is indistinguishable from attacker reconnaissance.
- If you only have a deployed site, audit the repository that produces it, or a copy of assets you are authorized to download.
- It does not produce working exploit payloads. It describes each vulnerability and its data flow in enough detail to fix it.

## What it detects

- Hardcoded or leaked secrets: API keys, private keys, tokens, and database connection strings with inline credentials.
- Dangerous sinks: `eval`, `new Function`, `innerHTML`, `dangerouslySetInnerHTML`, `document.write`, `child_process` calls, string-built SQL, and similar.
- Attacker-controllable sources: URL parameters, `postMessage` listeners, request bodies, client storage, cookies, and referrer.
- Cross-file vulnerable chains: data flowing from a source through transforms to a sink with no adequate sanitization, even when no single line looks alarming.
- Insecure configuration: disabled TLS verification, wildcard CORS, open redirects, and prototype pollution patterns.
- Weak cryptography: MD5/SHA-1 and non-crypto-safe randomness used for security purposes.

## Contents

- `SKILL.md` — the skill definition and workflow.
- `scripts/collect_js.sh` — collects JS/TS files into line-numbered markdown bundles and writes an inventory.
- `scripts/scan.py` — static scanner that emits candidate findings as JSON.
- `references/analysis-playbook.md` — source/sink taxonomy, chain-tracing method, and severity rubric.
- `references/report-template.md` — the report structure used for the final output.

## Installation

1. Download the `js-security-analyzer.skill` file.
2. Open your Claude skills settings.
3. Add or upload the `.skill` file.
4. Confirm the skill appears in your available skills list.

## Usage

1. Place the code you want to audit in a local directory. For a GitHub repository you are authorized to review, clone it first:
   ```
   git clone https://github.com/yourorg/yourrepo.git
   ```
2. Ask Claude to analyze the path, for example: "Audit ./yourrepo for vulnerabilities with the JS security analyzer."
3. Claude runs the pipeline:
   1. Collects and normalizes the source into markdown bundles.
   2. Runs the static scan to produce candidate findings.
   3. Reads the bundles and candidates, confirms exploitability, and traces vulnerable chains.
   4. Writes a prioritized report with fixes.
4. Review the report, prioritizing any critical items first.

## Running the scripts directly

You can run the tooling outside Claude if needed.

- Collect source into bundles:
  ```
  bash scripts/collect_js.sh <target_path> <output_dir>
  ```
- Run the static scan:
  ```
  python3 scripts/scan.py <target_path> <output_dir>
  ```

Outputs are written under `<output_dir>`:

- `bundles/` — line-numbered markdown bundles of the source.
- `inventory.json` — every file with size, line count, and a minified flag.
- `scan_findings.json` — candidate findings to focus the deep analysis.

## Handling of secrets in output

- Detected secrets are masked in the report, for example `sk_liv***GH`, rather than printed in full, so the report is safe to share.
- The report includes the file, line number, and key type, which is sufficient to locate and rotate the credential.
- Placeholder values such as `your-api-key-here` and environment variable references are not reported as live secrets.
- The recommended action for an exposed credential is rotation, not just deletion, because version control history retains the original value.

## Exclusions and limitations

- `node_modules`, `dist`, `build`, `coverage`, and `vendor` directories are excluded by default to reduce noise.
- Minified and bundled files are listed in the inventory but not audited, since they are not meaningfully reviewable. Audit the original source instead.
- Static analysis produces candidates, not confirmed issues. The deep-analysis step decides what is real and may downgrade or drop false positives.
- Findings are reported with a confidence level. Items that need manual confirmation are marked as such rather than overstated.
- Dependency CVE scanning is not included by default. It can be added as a separate step using `npm audit` or an equivalent tool.

## Requirements

- A Bash shell and standard Unix utilities (`find`, `wc`, `nl`, `awk`).
- Python 3.

## Notes

- The skill is designed for defensive use: finding and fixing issues in your own code before someone else does.
- Do not point it at code you are not authorized to review.
