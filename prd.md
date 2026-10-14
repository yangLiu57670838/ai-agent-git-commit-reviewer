# PRD: GitDiff Scout for Axios

## 1. Summary

GitDiff Scout is a command-line LLM agent that reviews a Git commit in the public `axios/axios` repository and identifies documentation or test files that may need updating. It explains each finding with evidence from the diff and the repository.

The project is a two-week research prototype. Its central question is:

> Does repository-aware tool use improve an LLM's ability to detect documentation drift compared with giving the model only a Git diff?

## 2. Problem

Small API changes are easy to merge without updating public documentation, examples, changelogs, or tests. Code review can catch this, but reviewers must understand both the change and a large repository's documentation layout.

Axios is a suitable corpus because implementation, tests, API documentation, and release notes are all public and have a long commit history:

- `lib/` — implementation
- `test/` — automated tests
- `README.md` — API and usage documentation
- `CHANGELOG.md` — release notes

## 3. Users

Primary user: a JavaScript maintainer or reviewer who wants a quick, evidence-based prompt about files that may have been overlooked in a commit.

Secondary user: an AI engineer evaluating whether repository tools improve agent quality.

## 4. Goals

1. Accept a local Axios repository and a Git revision range.
2. Produce zero or more actionable findings about potentially stale documentation or missing tests.
3. Cite the changed code and the relevant repository file/line for every finding.
4. Compare a diff-only LLM baseline with the repository-aware agent on a manually labelled historical-commit dataset.
5. Deliver reproducible commands, data, results, and a short research report.

## 5. Non-goals

- Editing files, committing changes, creating pull requests, or publishing comments.
- Determining whether an implementation is correct or secure.
- Replacing human code review.
- Training or fine-tuning a model.
- Supporting arbitrary repositories in version one.

## 6. Core user flow

```text
User chooses a revision range
        ↓
CLI retrieves the Git diff and changed files
        ↓
Agent searches the local repository and reads relevant files
        ↓
Agent returns supported findings or “no finding”
        ↓
Evaluation script compares output with labelled ground truth
```

Example command:

```bash
gitdiff-scout review --repo ../corpus/axios --range HEAD~1..HEAD
```

Example output:

```text
HIGH  README.md may be stale
Reason: lib/core/Axios.js adds a public request option, but README.md has no entry for it.
Evidence: lib/core/Axios.js:84–95; README.md:420–465
Suggested action: document the option under Request Config.
```

## 7. Functional requirements

### CLI

- `review --repo <path> --range <git-range>` runs a single agent review.
- `--format text|json` selects human-readable or structured output.
- `evaluate --dataset <path>` executes the baseline and agent against labelled tasks.
- `report --run <results-path>` generates report-ready Markdown tables and qualitative example candidates from a saved run.
- The CLI must fail clearly when the repository is missing, the Git range is invalid, or the working tree cannot be read.

### Revision resolution

`HEAD` and relative references such as `HEAD~1` move whenever the repository changes, so they are resolved exactly once per review:

- At the start of a review, resolve both ends of `--range` to full commit SHAs with `git rev-parse --verify <rev>^{commit}`.
- Every tool call during that review uses the resolved SHAs, never `HEAD` or the original range text.
- JSON output and run metadata record the original range text and both resolved SHAs.
- Accept only two-dot ranges (`A..B`) in version one; reject three-dot ranges (`A...B`) with a clear message.
- Relative references follow the first parent (`HEAD~1` is the first parent of `HEAD`). Merge commits are rejected in version one.
- Reject a range whose base does not exist, such as `HEAD~1` on the repository's root commit, with a clear message.
- Reviews work on a detached `HEAD`; no branch is required.

### Agent tools

The agent may use only read-only local tools:

- `get_git_diff(range)` — returns the diff and changed-file list.
- `search_repo(query, path?)` — searches tracked repository content.
- `read_file(path, start_line?, end_line?)` — reads a bounded excerpt.
- `get_file_history(path, limit?)` — optional; retrieves relevant historical commits.

### Findings

Each finding must include:

- category: `documentation`, `test`, or `changelog`
- priority: `high`, `medium`, or `low`
- target file path
- concise explanation
- at least one diff/code citation and one target-file citation
- a suggested next action
- confidence from 0.0 to 1.0

The agent must return an explicit empty finding list when no update is warranted. It must not invent files or citations.

### Dataset and labels

- Use 30 historical Axios commits selected from a fixed revision window.
- Include approximately 10 documentation-update cases, 10 no-update cases, and 10 ambiguous/test-focused cases.
- Store the commit SHA, revision range, label, expected file paths, and short human rationale in JSONL. Ranges use full SHAs (`<parent-sha>..<commit-sha>`), never `HEAD` or relative references.
- Exclude merge commits from the dataset.
- Document label definitions and inclusion/exclusion rules in `dataset/annotation_guide.md`.
- Keep a separate held-out test split of at least 10 tasks; use the remaining tasks as the development split for prompt and tool tuning.
- Record the Axios commit used to construct the dataset, model name, prompt version, and run timestamp.

## 8. Evaluation

Compare two conditions using the same model, temperature, prompt intent, and task set:

| Condition | Context and capability |
|---|---|
| Baseline | Commit diff only; no tools |
| Agent | Commit diff plus read-only Git/search/file tools |

Primary metrics:

- **Update decision precision and recall** — whether an update was needed.
- **Target-file accuracy** — whether the expected file was identified.
- **Unsupported-finding rate** — findings lacking valid evidence or incorrectly claiming a change is needed.

Report results overall and by category. Include 3–5 qualitative examples: correct cases, false positives, and false negatives.

Secondary metrics (reported, not optimised): tool calls per task, tokens per task, and latency per task for each condition.

Each evaluation run is saved under `results/runs/<timestamp>/` with per-task outputs and run metadata, so every number in the report can be traced to a specific run.

## 9. Technical approach

- Runtime: Python 3.11+.
- Interface: Typer or argparse CLI.
- Git access: `subprocess` with fixed, read-only commands; never use shell interpolation for revision ranges.
- Repository search: `rg` with bounded results.
- LLM layer: one tool-calling agent and one no-tools baseline using the same selected model.
- Outputs: Pydantic-validated JSON schema, then rendered as text.
- Tests: unit tests for Git parsing, revision resolution (`HEAD`, `HEAD~n`, root commit, merge commit, three-dot range, detached `HEAD`), tool bounds, JSON validation, and metric calculations.

Initial architecture:

```text
cli.py → review_service.py → agent.py / baseline.py
                              ├── tools/git_tools.py
                              ├── tools/repo_tools.py
                              ├── tools/path_guard.py
                              └── schemas.py
         eval/ → dataset.py → runner.py → scorer.py → report.py
                                              ↓
                          results/runs/<timestamp>/ → REPORT.md
```

Repository layout:

```text
src/gitdiff_scout/     CLI, agent, baseline, schemas, rendering
  tools/               read-only Git, search, and file tools
  eval/                dataset loading, run execution, scoring, report tables
scripts/               corpus setup, commit selection, figure generation
tests/                 unit tests
dataset/               labelled JSONL tasks and annotation guide
results/               saved runs and final metrics
reports/               experiment log and figures for REPORT.md
REPORT.md              final research report
```

## 10. Safety and guardrails

- The agent has no write, network, shell, or credential-access tools.
- Tool input paths must resolve inside the selected repository root.
- Limit search output and file excerpts to control cost and context size.
- Treat generated suggestions as review prompts, not authoritative patches.
- Use only public source code and commit metadata. Preserve upstream licence notices and attribute Axios as the corpus.

## 11. Research report

`REPORT.md` is the main research output. It must answer the central question in section 1 using only results from saved runs.

Required sections:

1. **Abstract** — question, method, headline result in one paragraph.
2. **Research question and hypothesis** — expected effect of tool use on each primary metric.
3. **Methodology** — corpus and pinned Axios commit, dataset construction and labelling, the two conditions, metrics, and run configuration (model, temperature, prompt version, timestamp).
4. **Results** — overall and per-category tables for baseline vs agent on the held-out test split, plus tool usage and cost.
5. **Qualitative examples** — 3–5 cases covering correct findings, false positives, and false negatives, each with the diff excerpt and model output.
6. **Discussion** — why the agent helped or did not, and observed failure patterns.
7. **Limitations and threats to validity** — small sample, label noise, possible model memorisation of public history, single repository.
8. **Next steps** — what to test with more time.
9. **Reproducibility** — exact commands to regenerate every table and figure.

Supporting material:

- `reports/experiment_log.md` — dated notes on prompt and tool changes and observations, kept throughout the two weeks.
- `reports/figures/` — charts generated by `scripts/make_figures.py`.
- Tables in `REPORT.md` are generated by the `report` command rather than typed by hand.

## 12. Acceptance criteria

The prototype is complete when:

1. A user can clone Axios, install dependencies, and run one review command from the README.
2. The agent returns schema-valid JSON and readable text for a valid revision range.
3. Every non-empty finding contains verifiable evidence citations.
4. The repository includes a labelled dataset, deterministic scoring script, and saved results.
5. `REPORT.md` contains every required section from section 11, compares baseline and agent performance on the held-out split, states limitations, and shows at least three examples.
6. Every table and figure in `REPORT.md` can be regenerated from a saved run with documented commands.
7. All project tests pass in CI or locally through one documented command.

## 13. Two-week plan

| Days | Outcome |
|---|---|
| 1–2 | Initialise repo, clone/pin Axios corpus, write annotation guide, select 30 commits. Start `REPORT.md` methodology draft and experiment log. |
| 3–4 | Implement revision resolution, Git/repository read tools, and a JSON output schema. |
| 5–6 | Build the tool-calling review agent and CLI text/JSON rendering. |
| 7 | Build and test diff-only baseline. |
| 8–9 | Label review, finalise train/test split, run initial experiments. Log observations. |
| 10–11 | Improve prompt/tool bounds; add tests and error handling. Build `report` command and figure script. |
| 12 | Run final evaluation on the held-out split and save all configuration/results. Freeze prompts after this point. |
| 13–14 | Generate tables and figures, write results, examples, discussion, and limitations in `REPORT.md`; polish README; record a short demo. |

Experiment log entries are added daily; the report draft grows alongside the code rather than being written only at the end.

## 14. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Labels rely on historical files changed, not necessarily files that should have changed. | Manually review every label and record rationale. |
| The agent has too much context and cost grows. | Set strict output/excerpt limits and log tool usage. |
| A commit mixes unrelated changes. | Exclude large, generated, dependency-only, or bulk-formatting commits. |
| Model memorised public repository history. | Frame results as a tool-use study, use recent commits where possible, and disclose this limitation. |
| Evaluation set is too small for strong claims. | Describe results as exploratory; publish the dataset and per-task outputs. |
| Prompts are tuned on the test split, inflating results. | Tune only on the development split; freeze prompts before the final held-out run. |
| Report is written from memory at the end and misses details. | Keep `reports/experiment_log.md` daily and generate tables from saved runs. |
| `HEAD` moves during or between runs, so results cannot be traced to a commit. | Resolve ranges to full SHAs once per review, store SHAs in the dataset, and record resolved SHAs in every output. |

## 15. Deliverables

- Source code and automated tests.
- `dataset/axios_commits.jsonl` and `dataset/annotation_guide.md`.
- `results/final_metrics.json` and per-task model outputs under `results/runs/<timestamp>/`.
- `REPORT.md` following the structure in section 11.
- `reports/experiment_log.md` and `reports/figures/`.
- `README.md` with setup, usage, and reproducibility instructions.
