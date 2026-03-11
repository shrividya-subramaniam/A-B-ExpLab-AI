# A/B ExpLab AI
#### Autonomous A/B Experimentation Agentic Engine


## Project Overview

A/B ExpLab AI transforms A/B testing from a manual, error-prone process into an automated statistically strong pipeline. Using a multi-agent system, ExpLab AI validates the integrity of the experiment, applies the appropriate statistical methods automatically, flags data quality risks in real-time, and translates complex results into actionable insights, while having a mandatory Human-in-the-Loop (HITL) safety net when data quality is compromised.

**Problem:** A/B tests fail silently when Sample Ratio Mismatches go undetected, the wrong statistical tests get applied to binary metrics, multiple comparisons inflate false discovery rates, and p-values get reported without business context. This leads to confident but wrong shipping decisions.

**Solution:** A pipeline of AI agents that gate validity and ensure integrity of the experiment before any statistics execution, apply assumption-checked frequentist and Bayesian tests, correct for multiple comparisons, and synthesize an actionable recommendation with risk assessment while pausing for human approval whenever data integrity is in question.

**The three failure modes it eliminates:**
- **Sample Ratio Mismatch** - broken randomization producing unreliable results
- **Wrong test selection** - applying parametric tests to binary retention metrics
- **False discovery inflation** - testing multiple metrics without correction



## Key Results

| Metric | Value |
|:---|:---|
| **Dataset** | Cookie Cats (90,189 users) |
| **Agents** | 4 — Gatekeeper, Statistician, Narrator, Supervisor |
| **Statistical Methods** | Chi-square, Welch's t-test, Mann-Whitney U, Bayesian Beta-Binomial |
| **Multiple Testing Correction** | Bonferroni + Benjamini-Hochberg (FDR) |
| **HITL Accuracy** | 100% (triggers correctly on SRM or invalid metric) |
| **Data Formats** | CSV, Parquet (10–100× faster), SQLite |
| **Visualization** | HTML bar charts saved as ADK Artifacts |

### Cookie Cats Findings

| Metric | Control (gate_30) | Treatment (gate_40) | Lift | Significant? |
|:---|:---|:---|:---|:---|
| `retention_1` | 44.82% | 44.23% | −0.59% | Borderline |
| `retention_7` | 19.02% | 18.20% | −0.82% | **Yes — p < 0.05** |

**Verdict: DO NOT SHIP ❌** — Gate 40 significantly hurts 7-day retention. The Bayesian posterior (~99% probability that gate_30 wins) corroborates the frequentist result.

---

## Quick Start

**Prerequisites:** Python 3.10+, a Gemini API key from https://aistudio.google.com/apikey

```bash
pip install google-adk pandas numpy scipy statsmodels python-dotenv
```

Create a `.env` file in the project root:
```bash
GEMINI_API_KEY=your_gemini_api_key_here
```

Download `cookie_cats.csv` from the [Kaggle dataset page](https://www.kaggle.com/datasets/mursideyarkin/mobile-games-ab-testing-cookie-cats) and place it in the working directory.

**Run the full agentic pipeline**
```python
import asyncio
await run_complete_analysis(file_path="cookie_cats.csv", metric="retention_7")
```

**Run the standalone statistical report (no agents required)**
```python
run_cookie_cats_analysis("cookie_cats.csv")
```

**Launch the ADK Web UI** (run all notebook cells first to generate the package):
```bash
python -m google.adk.cli web --port 8001
```
Then navigate to `http://localhost:8001` and enter:
```
Analyse the gate placement test (gate_30 vs gate_40) on retention_7.
Use file_path='cookie_cats.csv' for all tool calls.
```



### Agent Pipeline

```
Supervisor Agent
├─► Gatekeeper Agent   — SRM detection · metric alignment · power analysis
├─► Statistician Agent — assumption checking · frequentist + Bayesian tests · multiple testing correction
└─► Narrator Agent     — business verdict · risk assessment · HTML chart artifact
```

> **Full architecture explanation, agent design principle, and step-by-step code walkthrough are documented in the notebook itself.**  
> Open `AB_ExpLab_AI.ipynb` and read the markdown cells above each code section.



## Dependencies

```
google-adk>=1.23.0
pandas>=2.3.3
numpy>=2.3.5
scipy>=1.16.3
statsmodels>= 0.14.5
python-dotenv>=1.1.0
```


## Known Limitations

- **In-memory sessions** - session is lost on restart; production HITL workflows need persistent storage
- **Simulated HITL in notebook** - auto-approves in demo mode; deploy via `adk web` for real approval flows
- **Cookie Cats schema** - column names are hardcoded; adapting to other datasets requires updating tool functions
- **Binary-only Bayesian model** - Beta-Binomial applies only to binary metrics; continuous metrics need a different model



## References

- Benjamini, Y. & Hochberg, Y. (1995). Controlling the False Discovery Rate. *Journal of the Royal Statistical Society, Series B*, 57(1), 289–300.
- Kohavi, R., Tang, D., & Xu, Y. (2020). *Trustworthy Online Controlled Experiments.* Cambridge University Press.
- [Google ADK Documentation](https://github.com/google/adk-toolkit) · [Gemini API](https://ai.google.dev/docs)

