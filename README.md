# Automated Drought Impact Extraction

Everything runs from a single notebook: **`drought_pipeline.ipynb`**.

## Setup (once)

```bash
pip install -r requirements.txt
python -m spacy download de_core_news_lg    # and/or fr_core_news_lg, it_core_news_lg
```

The geoparser downloads its GeoNames gazetteer the first time it runs (a few GB, one
time only). The embedding model downloads about 1 GB the first time Block 8 runs.

## Folder layout

```
my_project/
├── drought_pipeline.ipynb
├── requirements.txt
│
├── input/
│   └── DE_articles.tsv          REQUIRED — your articles
│
├── annotation/
│   ├── DE_annotated_impacts_locations.csv   REQUIRED — span annotations
│   ├── DE_annotated_impacts_round2.csv      optional — more, added any time
│   ├── DE_annotated_irrelevant.csv          REQUIRED — irrelevant article IDs
│   ├── DE_annotated_relevant.csv            optional — extra relevant IDs
│   └── DE_annotated_foreign.csv             optional
│
└── output/                      created automatically
```

### Required columns

**Article file** (`.tsv` or `.csv`): `content_id`, `pubtime`, `head`, `subhead`,
`content`, `rubric`. Only `content_id`, `pubtime` and `content` are strictly required;
the rest are created empty if absent.

**Span annotations**: `FILE`, `ANCHOR`, `TAGS`, `QUOTE_TRANSCRIPTION`, and for locations
`VOCAB_LABEL`, `LAT`, `LNG`, `COMMENTS`.

**Irrelevant / relevant lists**: a `FILE` column of article IDs.

## Two modes: train once, then apply

Block 1 has a `MODE` switch:

- **`MODE = "train"`** — learn the models from a corpus you annotated (e.g. 1990–2024),
  then save them to `output/models/`. Do this once. Point `INPUT_FILE` at the file your
  annotations were made from.
- **`MODE = "apply"`** — run the saved models on new, un-annotated articles (e.g. 2025+).
  No annotations needed. Point `INPUT_FILE` at the new file.

The training-only blocks (4, 5a, 7a, 8a, 9) skip themselves automatically in apply mode,
so you can still run every cell top to bottom.

**Typical use:** run once in `"train"` mode on your historical corpus, then switch to
`"apply"` and run again for each new year of articles. You never re-process the old
articles.

## Running it

Open the notebook and run the cells from top to bottom. Only **Block 1** needs editing —
usually `MODE`, `LANGUAGE` and `INPUT_FILE`.

Every block saves its result, so you can stop after any block and continue later: re-run
Block 1 and Block 2 first, then carry on. The two geoparsing blocks resume automatically
if interrupted.

To process several languages, run the whole notebook once per language, changing
`LANGUAGE` and `INPUT_FILE` in Block 1.

## Adding more annotated data

1. Run **Appendix A** to export a fresh sample of un-annotated articles as TEI files.
2. Annotate them in your annotation tool.
3. Save the CSV export into `annotation/` with a name matching
   `{LANG}_annotated_impacts*.csv`, e.g. `DE_annotated_impacts_round3.csv`.
4. Re-run Block 4 onwards. All matching files are found and merged automatically, and
   rows appearing in several files are de-duplicated.

## Impact classifier: per-language training

The impact classifier (Block 8) is trained differently per language, all controlled from
`LANGUAGE_SETTINGS` in Block 1:

- **DE, FR** — the annotated articles are split into train / validation / test
  (`file_stratified`), and machine-translated ("synthetic") sentences from the other
  language are optionally added to the training set.
- **IT** — trained **entirely on synthetic data** (`no_real_train`). The real Italian
  data is used only for validation (threshold tuning) and test (scoring). Italian
  therefore *requires* its synthetic files to be present.

Synthetic files live in `SYNTHETIC_DIR` (default `classification/`) and are plain CSVs
with columns `sentence_raw, labels, file, sentence_relative_position`. Which files each
language uses is set by `impact_synthetic_files` in `LANGUAGE_SETTINGS`. Turn the whole
mechanism off with `USE_SYNTHETIC = False` (not possible for IT, which has no real
training data).

In **apply** mode the synthetic files are not needed — only the saved model is used.

## Main outputs

| File | Contents |
|---|---|
| `{LANG}_05_filtered.csv` | Articles that survived filtering |
| `{LANG}_09_impact_predictions.csv` | One row per impact sentence, with impact types and confidences |
| `{LANG}_11_location_clusters.csv` | One row per main location per article, with coordinates and canton |

Join the last two on `file` to get impacts with places.

## Turning geoparsing off

Geoparsing is on by default and is what makes the results Swiss-specific. Setting
`RUN_SWISS_FILTER = False` or `RUN_LOCATION_EXTRACTION = False` in Block 1 makes the run
much faster but keeps foreign droughts in the corpus and produces no locations. Useful
for a quick test run, not for final results.
