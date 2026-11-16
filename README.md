# QtFetcher (« Offre unique »)

<img src="images/mascot_240.jpg" align="right" width="140" alt="QtFetcher mascot, a magpie">

**Find the French public tenders where your company would face almost no competition.**

About 1 French public contract in 5 gets **a single bid**. For a small company that bids on public contracts, those tenders are the best opportunities: few or no competitors.
QtFetcher predicts which open or upcoming tenders are likely to get only one bid and matches them to the company's trade. It then says whether to go for each one, and why.

Built in one day at **Hackathon Open Data** (Hémicycle × Swanbase, Paris, 14 Nov 2026) by a team of two: [Jessica Falcon](https://github.com/jessicafalcon) and Nicolas Dormoy.
This is a copy of the team repo [jessicafalcon/offre-unique](https://github.com/jessicafalcon/offre-unique), with its full commit history.

## What it does

1. The company enters its **SIREN** (French company ID) or its name.
2. Its **profile** (trades as CPV codes, départements, contract sizes) is inferred from the public contracts it has already won.
3. It sees the matching tenders, **open now** (BOAMP) or **planned** (APProch), ranked by the predicted **chance of a single bid**.
4. Each tender gets a recommendation: **Aller** (go), **À étudier** (look closer) or **Passer** (skip). It comes with the facts behind it: the buyer's past contracts, the number of bids they received, and whether an incumbent keeps winning.
5. An LLM writes a two-sentence note for each tender. **Every number in the note is checked** against the computed facts, and a note that cites an unsupported number is rejected.

The interface is in French, since it is aimed at French SMEs.

## How it works

```
DECP (1.8 M past contracts, with bid counts) ──► features ──► LightGBM ──► calibrated P(single bid)
APProch (planned purchases, via Hémicycle API) ─┐                                   │
BOAMP (calls for tender open now)  ─────────────┴──► same features ──► score ───────┤
SIRENE (buyers, companies)  ──► company profile from its past wins ──► fit ─────────┴──► Streamlit app
                                                                                         + LLM note (number-checked)
```

- **Data**: DECP consolidated contracts (data.gouv.fr), APProch via the [Hémicycle](https://hemicycle.fr) API, BOAMP open data, and the SIRENE register. All are under the Licence Ouverte 2.0.
- **Model**: LightGBM on CPV category, amount band, duration, département, procedure, clauses, and out-of-fold target encodings of the buyer and category history. The split is by time (train 2023–24, test 2025–26), followed by isotonic calibration.
- **Recommendations**: rules on the calibrated ratio to the average, a past win in the same kind of contract and département, and the buyer's own single-bid history. Thresholds were picked on 2025 and reported on 2026.
- **Notes**: the LLM only gets computed facts. A checker rejects any note with a number that isn't in the facts, more than 45 words, or a changed recommendation.

## Results (test years 2025–2026, 320 k contracts)

| | AUC | Single-bid share among the top 10 % flagged |
|---|---|---|
| **Model (LightGBM)** | **0.694** | **49.6 %** |
| Baseline: buyer history | 0.639 | 42.2 % |
| Baseline: category rate | 0.630 | 38.0 % |

Base rate: 19.8 %. On 2026 contracts, tenders marked **Aller** got a single bid 41 % of the time, against 23 % for **À étudier** and 12 % for **Passer**.

### Experiments ([`experiments/`](experiments/))

- **Tender titles as a feature**: TF-IDF on titles raises AUC from 0.695 to **0.720** (+0.025, 95 % CI [+0.021, +0.029]). Sentence embeddings (via Featherless) add less: +0.005. → [`text_auc.md`](experiments/results/text_auc.md)
- **Matching tenders to a company**: a backtest asks whether the contracts a company actually won in 2026 would have been shown to it. Combining rules and embeddings raises recall@50 from 18.6 % to **22.4 %**. → [`matching_recall.md`](experiments/results/matching_recall.md)
- **Which LLM writes the notes**: Claude Haiku is compared with open-weight models (Gemma 4 31B, Qwen, gpt-oss-120B, GLM, Mistral) on the same facts, with automatic checks and an LLM judge run in both orders. The judge's verdict flipped with the order 25–47 % of the time, so its preferences are weak evidence. **Gemma 4 31B** passed 40/40 checks at about a tenth of Haiku's cost, so the app uses it. → [`notes_compare_v2.md`](experiments/results/notes_compare_v2.md), [`judge_v2.md`](experiments/results/judge_v2.md)

## Limits (also shown in the app)

- The score says how many bids a tender usually gets, **not whether you will win it**.
- 42 % of DECP contracts report a usable bid count, so the model learns from those.
- The map pin shows where the buyer is, not where the work happens. This matters for multi-site buyers such as the State, regions and central purchasing bodies.
- BOAMP notices under 90 k€ almost never carry a CPV code, so the open tenders shown are mostly above 90 k€.
- Following the hackathon rules, the app doesn't profile private individuals: it refuses sole traders and companies that opted out of the public register ("non-diffusibles").

## Run it

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # Hémicycle / LLM keys, only needed to refresh data or regenerate notes
python -m streamlit run app.py
```

Scored tenders, the model, calibration and the cached notes are in the repo. Three raw files are over GitHub's size limit: `data/raw/decp.parquet`, `data/clean/contracts.parquet` and `data/clean/awards.parquet`. To rebuild them and run the whole pipeline (pull → prepare → train → score → notes), follow [`docs/PIPELINE.md`](docs/PIPELINE.md). Hosting (Hugging Face Space, Docker) is in [`deploy/`](deploy/).

## Repo layout

```
app.py              Streamlit app (welcome, tenders, "Comment ça marche")
src/                pipeline: pull_*, prepare*, train, profiles, score, explain (LLM notes), buyers, siren
experiments/        title features, matching recall, LLM note comparison + results/
artifacts/          model, encoders, calibration, metrics, cached notes, benchmarks
tests/              fake-data pipeline test, stage_check.py (34 checks replaying the demo path)
deploy/             Dockerfile + Hugging Face Space bundle
docs/PIPELINE.md    full run order and developer notes
```
