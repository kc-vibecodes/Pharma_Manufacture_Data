# Escape from Flatland — does the patent record actually show it?

Testing a famous med-chem claim against 227,000 synthesis procedures pulled from pharmaceutical patents filed between 1976 and 2021.

## The question

In 2009, Lovering, Bikker and Humblet published *Escape from Flatland: Increasing Saturation as an Approach to Improving Clinical Success* (J. Med. Chem. 52(21), 6752–6756). The argument: medicinal chemistry had drifted toward flat, aromatic, sp2-heavy molecules because they are easy to make, and the more saturated, three-dimensional compounds were the ones surviving further down the clinical pipeline. They proposed **Fsp3** — the fraction of carbons that are sp3-hybridised — as a cheap one-number proxy for three-dimensionality.

It is a heavily cited paper. This repo asks whether it left a fingerprint on what industry actually filed, and what that shift cost on the bench.

Three questions, in order:

1. Did Fsp3 rise across the 1976–2021 window, and is there anything visible at 2009?
2. If molecules got more 3D, did that cost anything — lower yields, more elaborate routes?
3. How much of any trend is real chemistry versus an artefact of how the text was extracted?

**Caveat stated up front:** this is patent chemistry, which is not the same population as clinical candidates. Lovering's claim was about compounds that *succeed clinically*. Nothing here can speak to that. The most this can show is what got filed.

## Findings

| Test | Result |
|---|---|
| Median Fsp3, pre-2009 | 0.304 |
| Median Fsp3, post-2009 | 0.333 |
| Mann-Whitney U | p ≪ 0.001 (at n ≈ 227K, a very low bar) |
| Fsp3 vs. reported yield | r = 0.07 |
| Fsp3 vs. complexity score | r = 0.01 |

**The shift is real and small.** A ~10% relative move in median Fsp3, or roughly one carbon in thirty changing hybridisation on a typical molecule. Consistently signed, but small.

**Going 3D does not appear to cost yield.** The correlation with reported yield is not just negligible, it runs positive, which is the opposite of the "saturated targets are messier to get clean" intuition. If 3D chemistry is harder to make, it does not show up as a linear relationship at corpus scale.

**Survivorship is the most likely explanation for how flat the cost signal is.** Patents only record syntheses that worked well enough to file. Routes abandoned for terrible yield never make it in, which truncates exactly the tail that would show 3D chemistry being expensive.

## Data

Alvarado, Johnston & Brown, *Pharmaceutical manufacturing open datasets for machine learning in the primary and secondary domains* (CMAC, University of Strathclyde; SSRN 6394198). Two NLP-extracted sets from USPTO patent text:

- **Primary manufacturing (drug substance)** — 385,293 procedures across `procedures`, `operations`, `op_materials`, `op_conditions`, `materials_reference`. One row per synthesis procedure at the `procedures` grain. This is the set the analysis uses.
- **Secondary manufacturing (drug product)** — 9,216 records, loaded only as a row-count sanity check against the paper's stated figures.
- **Patent number → year crosswalk** — derived from USPTO *Issue Years and Patent Numbers, Since 1836*.

Expected layout:

```
./Data/
  procedures.csv
  operations.csv
  op_materials.csv
  op_conditions.csv
  materials_reference.csv
  sm_procedures.csv
  sm_apis.csv
  sm_operations.csv
  sm_op_materials.csv
  sm_op_conditions.csv
  patent_number_year_crosswalk.xlsx
```

## Method

**Complexity score.** There is no "how hard was this to make" column, so one is constructed. TF-IDF borrowed from text retrieval (Spärck Jones, 1972): each procedure is a document, each standardised action / condition / material is a term. Something appearing in almost every patent (`stir`, `add`, dichloromethane) carries almost no information; something appearing in a handful does. Summing IDF over a procedure's distinct terms rewards *unusual* chemistry rather than merely *long* chemistry.

IDF is computed at the **patent** level, not the procedure level. One patent often contains dozens of near-identical procedures, and counting documents as procedures would make a technique appearing 40 times inside one filing look 40× more common than it is.

Three channels (actions, conditions, materials), each min-max normalised before summing so no channel dominates on scale alone.

**Stub filter.** Many "procedures" are extraction artefacts — a sentence fragment that parsed into one or two operations with no materials and no conditions. Those are not short syntheses, they are parse failures, and they would drag every mean down. Anything with ≤2 operations *and* zero real materials *and* zero conditions is dropped.

**Yields.** `yield_percentage` arrives as a stringified list (multiple yields reported per procedure), parsed with `ast.literal_eval`, taking the max on the convention that the headline yield belongs to the titular product. Anything >100% is chemically impossible and is dropped rather than clipped — a 340% yield means the parser grabbed the wrong number, and clipping it to 100 would smuggle a bad record in as a good one.

**Fsp3 and stereochemistry.** `Descriptors.FractionCSP3` from RDKit, computed off `target_InChI`. Chiral centre count (`Chem.FindMolChiralCenters`) is pulled alongside it because Fsp3 on its own is blunt: a long floppy alkyl chain is all-sp3 and scores high without being three-dimensionally interesting. Stereocentres are the better signal for defined 3D structure, so the two curves are plotted together. Heavy atom count is also captured, to check later whether any Fsp3 trend is molecules getting *bigger* rather than rounder.

**Patent number → year.** Patent numbers are roughly chronological but not linear — the USPTO issues a wildly variable number per year, so twenty years of the 1980s covers less numeric ground than eight years of the 2010s. Binning by "patent number decade" would smear the time axis badly. The crosswalk gives the first utility patent number issued each year, turning this into an interval lookup (`searchsorted` on the boundaries, take the year to the left).

**Testing.** Fsp3 is bounded [0,1] and badly non-normal (a spike at 0 for fully aromatic compounds, another lump around 0.3–0.5), so Mann-Whitney U rather than a t-test. At n ≈ 227K essentially any non-zero difference lands at p < 0.001, so both medians are printed next to every p-value — the effect size is the number worth reading.

**Smoothing.** 5-year centred rolling mean, with raw yearly points drawn underneath at low alpha so the smoothing is visible rather than hidden. Years with fewer than 50 procedures are dropped; the early and late edges of the corpus are thin and produce meaningless spikes.

## Next

- Changepoint detection or interrupted time-series with a lag term, replacing the pre/post-2009 split.
- Spearman alongside Pearson for the yield association — the data is not bivariate normal.
- Regress Fsp3 on heavy atom count, to separate size from shape.
- Within-therapeutic-area cut to control for corpus composition.

## Running it

First, download the original datasets mentioned in file 'flatten_pm_daatset.ipynb'. Unzip the contents into a folder of your choosing. This is a huge file so give it ~15 mins to finish unzipping.

Next, download the required libraries -
```bash
pip install pandas numpy scipy matplotlib seaborn rdkit openpyxl tqdm
jupyter lab flatland_analysis_1.ipynb
```

In a separate location, generate the table CSVs using file 'flatland_pm_dataset.ipynb'. The uncompressed data is even larger, this step could take ~1hr to complete.

Download and place file 'patent_number_year_crosswalk.xlsx' at the same location as the CSVs you just created.

Lastly, run each cell in 'flatland_analysis_with_markdown.ipynb' in order.

## Sources

- Lovering, Bikker & Humblet (2009), *Escape from Flatland*, J. Med. Chem. 52(21), 6752–6756. [10.1021/jm901241e](https://doi.org/10.1021/jm901241e)
- Lovering (2013), *Escape from Flatland 2: complexity and promiscuity*, Med. Chem. Commun. 4, 515.
- Alvarado, Johnston & Brown, *Pharmaceutical manufacturing open datasets for machine learning in the primary and secondary domains*, CMAC / University of Strathclyde, SSRN 6394198.
- USPTO, *Issue Years and Patent Numbers, Since 1836* — basis for the number → year crosswalk.
- Spärck Jones (1972), *A statistical interpretation of term specificity and its application in retrieval*, Journal of Documentation 28(1) — the IDF weighting behind the complexity score.
- RDKit: `Descriptors.FractionCSP3`, `Chem.FindMolChiralCenters` — rdkit.org/docs
