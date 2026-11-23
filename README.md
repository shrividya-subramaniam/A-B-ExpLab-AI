   # A/B ExpLab AI

**Agentic A/B Test Analysis Engine**

## Project Overview

A/B ExpLab AI analyses a finished A/B test with a multi-agent pipeline built on Google ADK. Deterministic Python tools validate the experiment and compute every statistic; LLM agents decide which tools to call and explain the outcome. A validation gate, enforced in code, stops the pipeline for a human decision whenever data quality is in doubt.

**Problem:** A/B tests can fail silently. A Sample Ratio Mismatch goes unnoticed, several metrics are tested without correction, or a p-value is reported without the size of the effect. Each of these leads to confident but wrong shipping decisions.

**Solution:** Before any outcome statistics run, the pipeline checks the experiment for:

- **Sample Ratio Mismatch:** a group split that chance alone would rarely produce, a sign that assignment or logging is broken
- **Metric alignment:** whether the metric suits the type of test
- **Sample size:** whether the groups are large enough to detect the minimum effect of interest

Any issue pauses the pipeline until a person replies `APPROVED` or `REJECT`. After validation, the test is chosen by a recorded rule, several metrics are corrected for multiple comparisons, and the result is turned into a recommendation with a risk assessment.

## Key Results

| Item | Value |
|:---|:---|
| Dataset | Cookie Cats (90,189 users, mobile game A/B test) |
| Agents | 4 (Gatekeeper, Statistician, Narrator, Supervisor) |
| Statistical methods | Chi-square, Welch's t-test, Mann-Whitney U, Bayesian Beta-Binomial |
| Multiple testing correction | Bonferroni, Holm, Benjamini-Hochberg (FDR) |
| Validation gate | SRM, metric alignment and sample size; enforced in code, not by a prompt |
| Evaluation | 9 scenarios through the real agents; all passed every check with no code backstop. Each ran 3 times, but at temperature 0 the repeats were identical, so this is 9 distinct results, not 27 independent ones |
| Data formats | CSV, Parquet, SQLite, all through one loader |
| Visualisation | HTML report card, shown in the notebook and saved as an ADK Artifact |
| Model | Gemini 2.5 Flash-Lite at temperature 0; the Supervisor has a 512-token thinking budget |

## Cookie Cats Findings

The experiment moved the first gate in the game from level 30 (control, `gate_30`) to level 40 (treatment, `gate_40`). `retention_7` is the primary metric.

| Metric | Control (gate_30) | Treatment (gate_40) | Difference | Relative change | 95% CI of difference | p-value | Bonferroni p | Significant? |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|
| `retention_1` | 44.82% | 44.23% | −0.59 pp | −1.32% | −1.24 to +0.06 pp | 0.0755 | 0.1510 | No |
| `retention_7` (primary) | 19.02% | 18.20% | −0.82 pp | −4.31% | −1.33 to −0.31 pp | 0.0016 | 0.0032 | **Yes** |

pp = percentage points. p-values are from the chi-square test on the 2×2 table.

**Verdict: DO NOT SHIP gate_40.** Moving the gate to level 40 lowers 7-day retention, and the result stays significant after correcting for two metrics.

Three caveats:

- **Sample Ratio Mismatch warning.** The treatment group has 789 more users than the control group (chi-square p = 0.0086), an imbalance chance alone would produce less than 1% of the time. It is below the stricter p < 0.001 threshold, so the pipeline asks for human approval and reports the result with this caveat rather than discarding it.
- **The Bayesian result is not a second confirmation.** With a flat prior it is computed from the same counts and necessarily agrees: a 99.9% posterior probability that gate_30 retains better at 7 days.
- **Whom the effect applies to.** Many players stop before level 30 and never see either gate, so −0.82 pp is an average over all installers (an intention-to-treat effect); the effect on players who reach the gate is larger.

## Quick Start

**Prerequisites:** Python 3.10 or later (tested with 3.13) and a Gemini API key from https://aistudio.google.com/apikey.

```bash
pip install google-adk pandas numpy scipy statsmodels pyarrow python-dotenv
```

Create a `.env` file in the project root and keep it out of version control:

```
GEMINI_API_KEY=your_gemini_api_key_here
```

Download `cookie_cats.csv` from the [Kaggle dataset page](https://www.kaggle.com/datasets/mursideyarkin/mobile-games-ab-testing-cookie-cats) and place it in the working directory.

Open `AB_ExpLab_AI.ipynb` and run the cells in order. After that, these calls are available in the notebook:

```python
# Agent pipeline. On Cookie Cats it pauses at the SRM warning and asks you to type APPROVED or REJECT.
await run_complete_analysis(file_path="cookie_cats.csv", metric="retention_7")

# Standalone statistical report on both metrics, no LLM involved
run_cookie_cats_analysis("cookie_cats.csv")

# Checks of the tools and the validation gate on synthetic data (no API key needed)
run_sanity_checks()

# Agent evaluation: 9 scenarios x 3 repeats, about 5 to 10 minutes
await run_evaluation(n_repeats=3)
```

Without an API key the standalone report and the sanity checks still run; the agent pipeline, the evaluation and the Web UI need the key.

### ADK Web UI

The notebook's Web UI section writes the `explab_adk/` package from the notebook's own cells, then starts the server in the background with `start_adk_web()` (port 8001). Open http://localhost:8001, select `explab_adk` and send:

```
Analyse an A/B test. Test type: gate_placement. Variants: gate_30 (control) vs gate_40 (treatment). Metric: retention_7. File path: cookie_cats.csv (use this file_path for all tool calls).
```

The Supervisor pauses for approval because of the SRM warning; reply `APPROVED` or `REJECT`. The Artifacts tab then shows the HTML report, and the State tab shows the gate checks and stored results. Run `stop_adk_web()` when you are done.

## Agent Pipeline

```
Supervisor Agent
├─► Gatekeeper Agent   — SRM detection · metric alignment · sample size check
├─► Gate               — statistics stay blocked until the checks pass or a human approves
├─► Statistician Agent — assumption check · frequentist and Bayesian tests · multiple testing correction
└─► Narrator Agent     — business verdict · risk assessment · HTML report artifact
```

Design principles:

1. **The gate is enforced in code.** The statistical tools refuse to run until all three checks have passed or a human has approved the issues they raised.
2. **Numbers never travel through LLM text.** Tools store results in session state, and the headline figures in the final reply are written by code.
3. **Only a human can approve.** Approval is recorded by a callback that reads the user's own message; no tool can set it.
4. **Agent lapses are caught in code and logged.** If an agent goes silent, skips validation or forgets to ask for approval, code takes the next step from session state.

The full architecture and a step-by-step walkthrough are in the markdown cells of `AB_ExpLab_AI.ipynb`.

## Dependencies

```
google-adk>=1.23.0
pandas>=2.3.3
numpy>=2.3.5
scipy>=1.16.3
statsmodels>=0.14.5
pyarrow
python-dotenv>=1.1.0
```

## Known Limitations

- **A fixed workflow does not need an LLM orchestrator.** The steps are always the same, so a production version would run them in code and use an LLM only for the narrative. The agent design is kept to demonstrate orchestration and what it takes to make it reliable.
- **The evaluation covers one setup.** The clean result holds for Gemini 2.5 Flash-Lite at temperature 0 on nine scenarios, and claims without numbers in the model's text are not checked.
- **In-memory services.** Sessions, memory and artifacts are lost on restart; a pause that lasts hours needs a persistent session service and authentication for approvers.
- **Dataset-specific schema.** Column names and variant labels are constants in the notebook's Configuration cell; another dataset needs them changed.
- **Two variants, user-level metrics, one look.** No support for more than two variants, ratio metrics, variance reduction (e.g. CUPED), segment analysis or sequential testing.
- **Higher is assumed to be better**, which holds for retention but not for every metric.
- **Binary-only Bayesian model.** The Beta-Binomial model applies to binary metrics; continuous metrics would need a different model.

## References

- Benjamini, Y. & Hochberg, Y. (1995). Controlling the False Discovery Rate. *Journal of the Royal Statistical Society, Series B*, 57(1), 289–300.
- Kohavi, R., Tang, D., & Xu, Y. (2020). *Trustworthy Online Controlled Experiments*. Cambridge University Press.
- Google ADK documentation · Gemini API documentation
