---
noteId: "870cc250733411f1a77ca1722bd9b66d"
tags: []

---

# Near-Duplicate Detection on arXiv Abstracts

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/FraSeb6/Similarity_Ish_Abstract/blob/main/code.ipynb)

Finding pairs of **almost-identical paper abstracts** among millions of arXiv papers — quickly, with
**Shingling + MinHash + LSH**, and without ever comparing all pairs. Everything lives in one
notebook, `code.ipynb`, written to be read top to bottom.

## Results first

On a subsample of **49,327 abstracts** (character 9-shingles):

- **380** candidate pairs surfaced in a couple of seconds — out of ~1.2 billion possible pairs.
- **63.2% precision** over all candidates and **85.7% recall** against an exact ground truth.
- **337** duplicate groups; the largest is a real *"series of seven papers"*
  (arXiv `0710.4022`–`0710.4028`) sharing almost the same text.
- MinHash scales **linearly**: ~58 s on 50k abstracts, ≈50 min extrapolated to the full ~3M.

## Why it works (the 30-second version)

Two abstracts are "similar" when their sets of text fragments overlap a lot — that overlap is the
**Jaccard similarity**. Computing it for every pair is hopeless, so:

- **MinHash** squeezes each abstract into 100 numbers whose rate of agreement *estimates* Jaccard;
- **LSH** only ever compares abstracts whose signature chunks collide, discarding the obvious
  non-matches up front.

## The notebook, section by section

| # | Section | What happens |
|---|---|---|
| 0 | Setup | global config, including the `USE_SUBSAMPLE` switch |
| 1 | Ingestion | stream the 4 GB file line by line; drop sub-20-word abstracts |
| 2 | Shingling | each abstract → set of 9-character shingles, CRC32-hashed |
| 3 | MinHash | 100 hash functions → one 100-number signature per abstract |
| 4 | LSH | bands & rows **derived** from the target threshold → candidate pairs |
| 5 | Validation | precision **and** recall vs. a brute-force ground truth |
| 6 | Clusters | candidate pairs → graph → connected components = duplicate groups |
| 7 | Scalability | re-run on growing slices, confirm linear growth |

## Decisions I can defend at the exam

- **k = 9 characters, found by experiment.** With `k = 5` the shingles were too common
  (`" the"`, `"tion"`, …), so MinHash collided unrelated papers and precision collapsed to a few
  percent — including one bogus "cluster" of 681 unrelated abstracts. `k = 9` makes shingles rare
  enough to be meaningful, pushing precision up to ~63%.
- **Bands and rows are computed, not guessed.** From `t ≈ (1/b)^(1/r)` with `b·r = 100`, the code
  searches for the split closest to the target `0.5`, landing on **20 bands × 5 rows**
  (actual threshold ≈ 0.55).
- **Both precision and recall.** Precision alone hides the pairs you *miss*; measuring recall against
  an exact O(n²) ground truth on a few thousand documents is exactly what exposed the `k = 5` problem.
- **Readable over clever.** MinHash is written close to its definition — a minimum over hashed
  shingles — rather than as an opaque vectorized trick, so every line is explainable.

## Run it yourself

Open `code.ipynb` in Colab via the badge above (it also runs locally). You need a free Kaggle API
token (Kaggle → *Settings* → *API* → *Create New Token*), then fill the placeholders in the
**Ingestion** cell:

```python
os.environ["KAGGLE_USERNAME"] = "your_username"   # kept as "xxxxxx" in the committed notebook
os.environ["KAGGLE_KEY"]      = "your_key"
```

Then run all cells. To process the **full ~3M dataset** instead of the subsample, flip a single line
in the Setup cell — `USE_SUBSAMPLE = False` — and the same code runs end to end (~50 min for MinHash).

## Notes

- Dataset: [arXiv metadata](https://www.kaggle.com/datasets/Cornell-University/arxiv), CC0-1.0
  (Cornell University). Never commit a real Kaggle key.
- Project for the *Algorithms for Massive Data* course (MSc Data Science).
