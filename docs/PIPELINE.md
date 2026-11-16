# Offre unique: pipeline and developer notes

*Working notes written during the hackathon (14 Nov 2026). The overview is in the [README](../README.md).*

Upcoming French public tenders where a company would face little competition.
A model trained on past contracts (DECP) predicts the chance that each upcoming purchase
(APProch) gets a single bid. A company enters its SIREN, its profile is inferred from the
contracts it has won, and it sees matching tenders ranked by expected competition.

## Setup

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env        # then fill in the keys
```

## Run order

| Step | Command | Check before moving on |
|---|---|---|
| 0. Test the pipeline | `python tests/make_fake_data.py` then steps 2–7 | Everything runs. **Then delete `data/` before using real data.** |
| 1a. Upcoming purchases | `python src/pull_hemicycle.py approch` | ~106 pages = ~106 credits. Needs a **live** key (test keys stop at 100 requests/day). |
| 1b. Past contracts | Download the consolidated **parquet** from data.gouv.fr (search "DECP consolidées format tabulaire", ~240 MB) and save it as `data/raw/decp.parquet` | Free. Fallback: `python src/pull_hemicycle.py decp --since 2025-01-01 --max-pages 300` (costs credits, partial data). |
| 2. Prepare | `python src/prepare.py` | If it stops on missing columns, add the real spelling to `ALIASES` in `src/common.py`. |
| 3. Data check | `python src/check_data.py` | Read `reports/data_check.md`. Check the raw procedure labels map correctly, the APProch duration and amount formats parse, and the verdict. |
| 4. Train | `python src/train.py` | Model should beat both baselines on the test years. If not, say so in the pitch and use the buyer-history baseline as the score. |
| 5. Profiles | `python src/profiles.py` | Pick a demo company from `reports/demo_candidates.csv`, set `DEMO_SIREN` in `.env`. |
| 6. Score | `python src/score.py` | Top rows look plausible; spot-check two against decp.info. |
| 7. Notes | `python src/explain.py --test` then `python src/explain.py --top 10` | Uses Claude by default (`LLM_PROVIDER=anthropic`); set `LLM_PROVIDER=featherless` to switch. Pre-generates and caches notes for the demo company. Rejected notes (unsupported numbers) are not shown. |
| 8. App | `streamlit run app.py` | |

## What each file does

- `src/common.py`: paths, column aliases, and the features built identically for both datasets (CPV division, amount band from APProch's own bands, duration bin, département, procedure, clauses).
- `src/prepare.py`: one row per contract (de-duplicated on buyer + id), one row per winning company (for profiles), APProch with the same features.
- `src/train.py`: time split (train before `SPLIT_YEAR`, test from it), out-of-fold target encoding for buyer and category rates, LightGBM, two baselines, calibration, then a final model on all data.
- `src/profiles.py`: company profile from past wins, fit of each upcoming purchase, demo-company finder.
- `src/score.py`: probability, top 3 factors (LightGBM contributions), and the buyer's past contracts in the same CPV group: number of offers, incumbent and streak.
- `src/explain.py`: note from Claude or Featherless (LLM_PROVIDER), built from computed facts only; every number in the note must appear in the facts or the note is rejected.
- `app.py`: two tabs. "Appels d'offres" (SIREN, list, detail) and "Preuves" (model vs baselines, sources, limits).

## Known judgement calls (check on real data)

- Procedures "sans publicité ni mise en concurrence" are excluded from training: a single bid is expected there by design.
- Contracts with `offresrecues` of 0 or above 200 are treated as missing.
- "Comparable past contracts" means same buyer and same first 3 CPV digits, falling back to the first 2.
- The fit score weights are arbitrary (60 % CPV division, 30 % département, 10 % amount band).
- Buyer names come from DECP if the file has them; otherwise the SIREN is shown.

## Open calls for tender (BOAMP), optional second source

APProch lists *planned* purchases (often 2027–2029, mostly central State). BOAMP lists calls for tender that are
*open now*, including local authorities. Free API, no key.

```bash
python src/pull_boamp.py            # ~75 requests, prints CPV / SIRET extraction rates
python src/prepare_boamp.py         # keeps calls for tender with a CPV code
python src/train.py --variant boamp # same model without amount, duration and clauses (absent from BOAMP)
python src/score.py --source boamp  # -> data/clean/scored_boamp.parquet
```

The app, `find_demo.py`, `check_fit.py` and `explain.py` pick up both sources automatically.
Test without network: `python tests/fake_boamp_pull.py` (after `tests/make_fake_data.py`).

## Partner setup (hackathon, 14 Nov)

Read `CLAUDE.md` (same as `projet-brief.md`) first: it has the current state, demo script and decisions.

1. `python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
2. `cp .env.example .env` and ask Jessica for the keys (never commit `.env`). Set `DEMO_SIREN=834709248`.
3. Get the three files that are too big for GitHub and put them at the same paths:
   `data/raw/decp.parquet`, `data/clean/contracts.parquet`, `data/clean/awards.parquet`.
   Fastest: AirDrop from Jessica's laptop. Otherwise download DECP (see step 1b above) and run `python src/prepare.py`.
4. `python -m streamlit run app.py`

Everything else (model, calibration, notes, scored tenders) is in the repo. Work on a branch and open a PR; don't push to `main` during the demo prep.

## For the UI: buyer type and map (added 14 Nov, 13:00)

`load_all_scored()` now adds, for every tender: `buyer_type` (Commune, Hôpital santé, Collège lycée, Région, État ministère…),
`buyer_lat`, `buyer_lon`, `buyer_address`, `buyer_multi_site` and `buyer_location_source`.
Source: public company register (SIRENE via recherche-entreprises.api.gouv.fr), built by `python src/buyers.py`
into `artifacts/buyers.json`. **The pin is where the buyer is, not the worksite.** For multi-site buyers
(State, regions, départements, Ville de Paris, central purchasing bodies) say so instead of implying a precise place.

Paste-ready snippet for `app.py`, after `t = relevant(...)`:

```python
from explain import recommend_row
t["reco"] = [recommend_row(r, prof) for _, r in t.iterrows()]

types = sorted(t["buyer_type"].dropna().unique())
pick = st.multiselect("Type d'acheteur", types)
if pick:
    t = t[t["buyer_type"].isin(pick)].reset_index(drop=True)

COLORS = {"Aller": "#1a9850", "À étudier": "#fdae61", "Passer": "#bdbdbd"}
m = t.dropna(subset=["buyer_lat", "buyer_lon"]).assign(color=lambda d: d["reco"].map(COLORS))
st.map(m, latitude="buyer_lat", longitude="buyer_lon", color="color", size=300)
st.caption("Point = adresse de l'acheteur (registre SIRENE), pas forcément le lieu des travaux. "
           f"{int(m['buyer_multi_site'].sum())} acheteurs multi-sites : position indicative.")
```

## For the UI: contract size, kind of purchase, two-stage procedures (added 14 Nov, 14:00)

`load_all_scored()` also adds:
- `taille` = Petit / Moyen / Grand, and `taille_marche` = the label to display ("90 k€ – seuils européens", "≈ 100k – 500k€").
  Open tenders (BOAMP): official BOAMP size bracket. Planned purchases (APProch): estimated amount band (Petit < 100 k€, Moyen < 1 M€, Grand ≥ 1 M€, approximate).
- `nature_achat` = Travaux / Services / Fournitures.
- `procedure_note` = a warning for restricted / negotiated / competitive-dialogue procedures (apply first, bid only if shortlisted), else empty.

```python
c1, c2 = st.columns(2)
sizes = c1.multiselect("Taille du marché", ["Petit", "Moyen", "Grand"])
kinds = c2.multiselect("Nature", ["Travaux", "Services", "Fournitures"])
if sizes:
    t = t[t["taille"].isin(sizes)].reset_index(drop=True)
if kinds:
    t = t[t["nature_achat"].isin(kinds)].reset_index(drop=True)
# in the tender detail, under the facts:
if isinstance(r.get("taille_marche"), str):
    st.caption(f"Taille : {r['taille_marche']}")
if isinstance(r.get("procedure_note"), str):
    st.warning(r["procedure_note"])
```
Known gap: BOAMP notices under 90 k€ almost never carry a CPV code (1 %), so the pipeline drops 461 of 466 of them; the open tenders shown are almost all above 90 k€.

## SIREN check (src/siren.py, added 14 Nov 15:15)

`clean_siren(raw)` keeps the first 9 digits (spaces, dots and a full SIRET accepted); `siren_valid(s)` checks the
Luhn key (La Poste 356000000 excepted) and rejects repeated-digit placeholders. The ~2.5 % of DECP company SIRENs
that fail it are typos (checked: not in the national register). To use it in `app.py`, replace the
`re.sub(...)` / `PLACEHOLDER_SIRENS` lines of the SIREN block with:

```python
from siren import clean_siren, siren_valid
siren = clean_siren(raw)
if len(siren) < 9:
    st.write("Un SIREN compte 9 chiffres (ce sont aussi les 9 premiers chiffres du SIRET).")
    st.stop()
if not siren_valid(siren):
    st.write("Ce numéro n'est pas un SIREN valide (sa clé de contrôle ne correspond pas). Vérifiez-le.")
    st.stop()
```
