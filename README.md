# DS-6015 GovSpend Price Variability Analysis

**Detecting price anomalies and benchmarking variability in public sector procurement data**

UVA School of Data Science · MSDS Capstone · Sponsored by [GovSpend](https://govspend.com)

---

## The problem in one paragraph

Public agencies record purchases with inconsistent units of measure. A case of gloves at \$120 and a box of the same gloves at \$12 are not comparable numbers until you know how many gloves are in each. Roughly a fifth of line items are priced in something other than "each," and those tend to be the higher-dollar rows. Until quantities mean the same thing, any outlier detector will mostly rediscover packaging differences instead of pricing problems.

This repository contains a pipeline that converts procurement line items to a per-item price, groups genuinely comparable products, and surfaces unusual prices as a ranked review queue.

---

## Headline results

| | |
|---|---|
| Line items processed | 917,321 |
| Records converted to per-item pricing | 43,130 |
| Records declined rather than guessed | 2,865 |
| Price variability before normalization (CV) | 0.653 |
| Price variability after normalization (CV) | **0.078** |

**Normalization works.** Across 58 product groups where the description was identical and only the pack size differed, converting to a per-item price reduced measured variability in every single group.

**But grouping was the real bottleneck.** Using the supplied category field as the grouping key, median within-group variability was 0.897. Grouping instead on description similarity brought it to 0.124, against a ceiling of 0.043 for exact-description matches.

| Grouping strategy | Median within-group CV |
|---|---|
| Category + unit of measure (supplied) | 0.897 |
| Description similarity (threshold 0.7) | **0.124** |
| Exact description match (ceiling) | 0.043 |

**Normalization changes conclusions, not just numbers.** Of the transactions flagged under each approach, only 73 appeared in both sets. 673 were surfaced only after normalization; 964 raw-price flags disappeared once comparability was established.

---

## How it works

```
Raw procurement extract  (917,321 line items)
        │
        ├── remove non-product rows ....... freight, tax, order summaries, spreadsheet errors
        │
        ├── classify unit of measure ...... already per-item │ time-based services │ containers
        │
        ├── rule-based pack parser ........ reads pack counts from the description and verifies
        │                                    the unit matches what was actually priced;
        │                                    declines when the evidence is ambiguous
        │
        ├── learned model ................. trained on parser output as weak supervision,
        │                                    predicts pack size for records the rules missed
        │
        ├── item grouping ................. description similarity, connected components
        │
        └── review queue .................. robust outlier scoring within comparable groups
```

### Design principle: decline rather than guess

If a description says `100/PK` but the row is priced by the case, `100` is items per pack, not items per case. Accepting it would understate the true quantity by the case multiple, and that wrong per-item price would flow into the benchmarks and reappear as a false anomaly. So the parser abstains. 2,865 records were declined on these grounds.

A declined record is merely absent from the analysis. A wrong conversion actively manufactures a false finding. We weight precision above coverage throughout.

---

## Evaluation

We evaluated against a hand-labeled sample of 300 records, stratified so that both converted and declined records could be assessed. Annotators worked from a blinded file that did not show the parser's output.

| | |
|---|---|
| Records labeled | 300 |
| Determinable from the description | 252 |
| Marked unknowable by the annotator | 46 |
| High-confidence cases scored | 102 |
| Medium-confidence cases scored | 34 |
| Unverifiable (parser answered, annotator could not) | 14 |
| Declines judged appropriate | 150 |
| Parser–annotator disagreements | 2 |

### Learned model

The parser's high-confidence output supplies training labels at no annotation cost. Two evaluations are reported, because the gap between them matters:

| Evaluation | Accuracy |
|---|---|
| Majority-class baseline | 11.8% |
| Random row split (optimistic) | 88.6% |
| **Product-level split (reported)** | **70.3%** |

The 28-point gap is the portion of the random-split score that came from seeing the same product in both training and test. We report the product-level figure. Classifier comparison on that split: LinearSVC 70.3%, SGD (modified huber) 69.3%, logistic regression 60.6%, ComplementNB 48.2%.

---

## Repository contents

```
notebooks/
  govspend_pipeline_results.ipynb   full pipeline: EDA, parser, model, grouping, evaluation
paper/
  main.tex                          IEEE-format final paper
  references.bib
  figures/
presentation/
outputs/                            generated artifacts (see note on data below)
```

### Running the pipeline

1. Place the merged procurement extract in the working directory.
2. Open `govspend_pipeline_results.ipynb` and set `CSV_PATH`.
3. Run all cells in order.

The notebook detects an existing merged extract and skips the rebuild step, and it will not overwrite an existing hand-labeled evaluation set.

**Requirements:** Python 3.10+, pandas, numpy, scikit-learn, scipy.

---

## Data availability

The procurement data is provided by GovSpend under a non-disclosure agreement and is **not included in this repository**. The code assumes a merged line-item extract with description, quantity, unit price, unit of measure, category, and total price fields.

---

## Known limitations

- **Coverage is partial.** Most item descriptions never state a package size in any recoverable form. Better parsing alone cannot lift this ceiling.
- **The category field is unreliable** at the line-item level and should not be used to group items for price comparison.
- **The unit of measure field does not always describe what was priced.** Some records coded as individual items carry a package count in the description and are priced per package.
- **Outlier output is a review queue, not a findings list.** At this stage it surfaces some of our own conversion errors alongside genuine price differences.
- **Single jurisdiction.** All records analyzed are Virginia transactions; recording conventions differ elsewhere.

---

## Team

| | |
|---|---|
| Daniel Ra | wr9cn@virginia.edu |
| Claudio Cela | cc5hu@virginia.edu |
| Jack Thompson Hays | jth2uyr@virginia.edu |
| Maneesh Reddy Malpeddi | kwc4md@virginia.edu |

**Sponsor:** Matt Beard, Senior Technical Project Manager, GovSpend
**Faculty mentor:** Phillip Waggoner, University of Virginia

---

## License

MIT — see [LICENSE](LICENSE).
