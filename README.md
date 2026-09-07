# SongExplicitness
Analyzing song explicitness as it relates to mathematical representations of musical fetaures. Made as final project for DSC80.

## Introduction
 
This project explores a dataset of Spotify tracks, each described by musical features such as
energy, danceability, tempo, loudness, and key, alongside metadata like popularity, genre, and
whether the track is marked explicit.
 
**Central question: Can we predict whether a song is explicit based on its musical features
alone — and is explicitness reflected in a track's underlying sound (e.g. its energy)?**
 
This matters because it gets at something intuitive but rarely tested rigorously: do explicit
lyrics *sound* different, or is explicitness purely a lyrical/content label independent of the
music itself?
 
The dataset contains **`<FILL IN: number of rows after cleaning>`** tracks. The columns most
relevant to this analysis are:
 
| Column | Description |
|---|---|
| `explicit` | Whether the track contains explicit content (the variable we're ultimately predicting) |
| `energy` | Perceptual intensity and activity (0–1); fast, loud, noisy tracks score high |
| `danceability` | How suitable the track is for dancing (0–1) |
| `loudness` | Overall loudness in decibels |
| `valence` | Musical positiveness (0–1); high = happy/cheerful, low = sad/angry |
| `tempo` | Estimated beats per minute |
| `key` | Musical key of the track (0=C, 1=C#, ..., 11=B) |
| `mode` | Major (1) or minor (0) key |
| `time_signature` | Estimated beats per measure |
| `track_genre` | Genre label, as tagged by Spotify |
 
---
 
## Data Cleaning and Exploratory Data Analysis
 
### Cleaning steps
 
- Dropped the `Unnamed: 0` index column, which carried no information.
- Removed a single corrupted row (`track_id == '1kR4gIb7nGxHPI3D2ifs59'`) that had missing
  `artists`/`album_name`/`track_name` and a `duration_ms` of 0 — clearly a blank/broken record
  rather than a real track.
- Deduplicated by `track_id`, keeping the first occurrence. Many tracks in the raw data appear
  multiple times because the same physical track is cross-tagged under several genres; since this
  project treats each track as a single observation rather than analyzing genre-level rows, this
  collapsing was appropriate.
- Replaced sentinel values that don't represent real musical measurements with `NaN`:
  - `key == -1` (Spotify's own "no key detected" code)
  - `tempo == 0` (a track cannot have 0 BPM — this reflects failed beat detection)
  - `time_signature == 0` (not a valid time signature)
These steps reflect the data generating process: Spotify's audio-feature pipeline algorithmically
estimates these values, and each sentinel marks a case where that estimation failed rather than a
true "zero" measurement.
 
Cleaned data preview:
 
```
<FILL IN: paste output of music_tracks.head().to_markdown() here>
```
 
### Univariate Analysis
 
<!-- Embed your duration_min histogram -->
<iframe
  src="assets/duration-distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
Track duration clusters heavily around 3 minutes, as expected for popular music, but with a
notable long tail — some of the longest tracks turn out to be classical pieces and compilations
(e.g. multi-movement recordings), which explains the extended right tail beyond typical pop-song
length.
 
<!-- Embed your energy histogram -->
<iframe
  src="assets/energy-distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
Energy increases fairly steadily across its range rather than peaking at a single typical value,
suggesting the dataset spans a genuinely wide mix of low-energy (acoustic, classical) and
high-energy (metal, dance) genres rather than being dominated by one style.
 
### Bivariate Analysis
 
<!-- Embed your loudness vs energy scatter plot -->
<iframe
  src="assets/energy-vs-loudness.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
Energy and loudness show a clear positive relationship, though the shape isn't perfectly linear —
it curves in a way reminiscent of logistic growth, suggesting the relationship saturates at high
loudness/energy levels.
 
<!-- Embed your percent-explicit-by-energy-bin bar chart -->
<iframe
  src="assets/pct-explicit-by-energy.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
The percentage of explicit tracks generally rises with energy, consistent with the hypothesis test
result below — but the very highest energy bin dips unexpectedly. Investigating further, the
highest-energy bin is dominated by metal subgenres (grindcore, death-metal, black-metal,
heavy-metal), which tend to be explicit at a lower rate than genres like hip-hop despite their high
energy — a reminder that the energy–explicitness relationship isn't uniform across genres.
 
### Interesting Aggregates
 
Grouping numeric features by `explicit` status:
 
```
<FILL IN: paste output of the groupby('explicit').mean() table .to_markdown() here>
```
 
Explicit tracks average higher energy, danceability, and speechiness, and lower acousticness and
instrumentalness than non-explicit tracks — consistent with explicit content skewing toward more
produced, vocal-forward, higher-intensity music.
 
---
 
## Assessment of Missingness
 
### MNAR Analysis
 
I believe **`tempo`** is likely **MNAR** (Missing Not At Random). Tempo is algorithmically
estimated by Spotify's beat-tracking pipeline, and a value goes missing specifically when that
algorithm fails to confidently detect a stable beat. That failure is driven by properties of the
track's *actual rhythmic structure* — e.g. highly ambient, atonal, or non-percussive music is
both harder to beat-track *and* would have had an unusual/ambiguous true tempo value if it could
be measured. In other words, the reason the value is missing is tied to the very quantity that's
missing, which is the defining feature of MNAR. Additional data that could help move this toward
MAR would be a confidence score from Spotify's beat-tracking algorithm itself, or raw audio access
to allow independent tempo estimation.
 
### Missingness Dependency
 
I tested whether the missingness of `tempo` depends on other columns using permutation tests with
Total Variation Distance (TVD) as the test statistic for categorical columns.
 
- **`tempo` missingness *depends on* `track_genre`**: observed TVD ≈ 0.19, p-value = 0.0. This
  makes sense — some genres (e.g. ambient, spoken-word-heavy genres) are inherently harder for a
  beat-tracker to analyze than others.
- **`tempo` missingness *does not depend on* `<FILL IN: popularity or explicit — rerun to confirm which>`**:
  p-value = `<FILL IN>`, indicating no detectable relationship — consistent with this column being
  unrelated to the audio-analysis pipeline that produces tempo estimates.
I ran the same pair of tests for `time_signature` missingness:
 
- **Depends on `track_genre`**: observed TVD ≈ 0.46, p-value = 0.0.
- **Does not depend on `mode`**: p-value = 0.207.
<!-- Embed a plot related to your missingness exploration -->
<iframe
  src="assets/missingness-plot.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>
---
 
## Hypothesis Testing
 
**Question:** Are explicit tracks more energetic on average than non-explicit tracks?
 
- **Null Hypothesis:** Explicit tracks are as energetic, on average, as non-explicit tracks; any
  observed difference is due to random chance.
- **Alternative Hypothesis:** Explicit tracks are more energetic, on average, than non-explicit
  tracks.
- **Test statistic:** mean(`energy` | explicit) − mean(`energy` | non-explicit)
- **Significance level:** 0.05
Using a permutation test (10,000 shuffles of the `explicit` label), the observed difference in
means was **0.092**, and **0 out of 10,000** permuted differences were as extreme, giving a
**p-value of 0.0**.
 
**Conclusion:** We reject the null hypothesis. This dataset provides strong evidence that explicit
tracks tend to be more energetic than non-explicit tracks. (As with any statistical test, this
doesn't *prove* the relationship — it means the observed gap is very unlikely to have arisen from
random chance alone under the null model.)
 
Interestingly, when this same test was repeated within just the hip-hop genre, the relationship
reversed: more energetic hip-hop tracks were actually *less* likely to be explicit, suggesting the
overall pattern is not uniform across genres and may partly reflect which genres tend to be
explicit in the first place, rather than a universal property of "explicit music."
 
---
 
## Framing a Prediction Problem
 
**Prediction problem:** Can we predict whether a track is explicit, based on its musical features
(key, mode, danceability, energy, loudness, valence, tempo, time signature)?
 
- **Type:** Classification (binary — explicit vs. not explicit)
- **Response variable:** `explicit`. I chose this because it's the central question motivating the
  whole project: whether explicitness is reflected in a track's sound.
- **Features used are all knowable at "time of prediction"**: all are Spotify's own
  algorithmically-derived audio features, generated from the track's audio at ingestion — none of
  them depend on post-release information like `popularity`, so there's no risk of leaking
  future/outcome information into the model.
- **Evaluation metric:** `<FILL IN based on Issue #3 above — recommend explaining you use accuracy
  but note the class imbalance (~91% not-explicit), and why you also report precision/recall/F1
  from the classification_report rather than relying on accuracy alone>`
---
 
## Baseline Model
 
**Features:** `key` (nominal), `mode` (nominal), `danceability` (quantitative), `energy`
(quantitative).
 
`key` and `mode` were one-hot encoded since they're categorical (key has no meaningful numeric
ordering; mode is a binary category). `danceability` and `energy` were left as-is since they're
already bounded, continuous quantitative measures. All steps were implemented in a single
`sklearn` `Pipeline`.
 
**Performance:** Accuracy = `<FILL IN after re-running with the corrected, matching test_size — see Issue #1>`
 
Whether this is "good": on its own it beats a majority-class baseline (~91.4% not-explicit), but
only modestly — this simple model has limited signal from just 4 features and doesn't yet use
several plausibly-relevant columns (loudness, tempo, valence, time signature).
 
---
 
## Final Model
 
**Model:** Random Forest Classifier (chosen over the baseline's Decision Tree because ensembling
many trees reduces the overfitting risk of a single deep tree, which is a decision tree's main
weakness).
 
**New engineered features (in addition to the baseline's 4):**
 
1. **`time_signature`, one-hot encoded** — even though it's stored numerically, time signature is
   categorical in nature (3/4/5 beats-per-measure aren't meaningfully ordered or additive), so
   encoding it as a category rather than a raw number avoids implying a false numeric relationship.
2. **`tempo`, median-imputed then quantile-transformed** — tempo has missing values (from failed
   beat detection, per the MNAR discussion above) that needed imputing, and is right-skewed with
   some extreme outliers; `QuantileTransformer` maps it to a uniform distribution so extreme tempo
   values don't disproportionately influence the model.
`loudness` and `valence` were added as additional plain quantitative features.
 
**Hyperparameters tuned:** `max_depth`, `n_estimators`, and `min_samples_split`, selected via
`GridSearchCV` with 5-fold cross-validation. These were chosen because tree depth and minimum
samples per split most directly control overfitting — a decision tree/forest's key weakness — while
the number of estimators controls how stable the ensemble's averaged predictions are.
 
**Best hyperparameters:** `<FILL IN final numbers after fixing Issue #1>`
 
**Performance:** Accuracy = `<FILL IN>`, compared to the baseline's `<FILL IN>` — an improvement
(once evaluated on the same held-out test set).
 
---
 
## Fairness Analysis
 
**Question:** Does the final model perform differently for hip-hop tracks versus dancehall tracks?
I chose this pair because earlier hypothesis testing revealed that the energy–explicitness
relationship behaves differently within hip-hop than in the dataset overall, making it an
interesting candidate for a fairness check.
 
- **Group X:** hip-hop tracks
- **Group Y:** dancehall tracks
- **Evaluation metric:** Precision
- **Null Hypothesis:** The model's precision is roughly the same for hip-hop and dancehall tracks;
  any difference is due to random chance.
- **Alternative Hypothesis:** The model's precision for hip-hop tracks is higher than for
  dancehall tracks.
- **Test statistic:** precision(hip-hop) − precision(dancehall)
- **Significance level:** 0.05
Observed hip-hop precision was 0.957 vs. dancehall's 0.857 (difference ≈ 0.099). A permutation
test (1,000 shuffles) gave a **p-value of 0.152**.
 
**Conclusion:** Since 0.152 > 0.05, we fail to reject the null hypothesis — there isn't strong
evidence that the model treats hip-hop and dancehall tracks differently in terms of precision, at
least by this test. (Note: hip-hop's overall accuracy on this model was notably lower — 0.807 —
than the model's global accuracy, suggesting that while precision parity holds, the model's
ability to correctly classify hip-hop tracks overall may still be weaker than for the dataset as a
whole.)