# Aura Peak: Sound Profile & Licensing Cost Analysis

**One-line hook:** Mega-hits aren't more energetic or danceable than mid-tier tracks (danceability −1.2%, energy −3.0%, valence −5.6%) — so Aura's "Aura Peak" curation algorithm shouldn't chase loudness, it should chase acoustic similarity to cheaper, less-saturated tracks instead.

## Business Context

Aura is an independent B2B music licensing and curation agency that builds dynamic playlist channels for gyms, retail chains, and hospitality clients on a subscription model. I'm the Senior Analyst on the Content Strategy & Curation team. This quarter Aura is launching **Aura Peak**, a new high-intensity channel for gyms and busy retail floors, and needs two things resolved before launch: (1) a data-backed blueprint for which audio features Elena's curation algorithm should filter on, and (2) whether Marcus's licensing team can substitute cheaper, lesser-known tracks for expensive mega-hits without losing the "vibe" clients are paying for.

## The Data

- **Source:** Spotify audio-feature + playlist export (`high_popularity_spotify_data.csv`)
- **Grain:** one row per track–playlist combination — a track appears once per playlist it's featured in
- **Volume:** 1,686 rows / 29 columns → 1,437 unique tracks after deduplication
- **Date range:** track release dates span historical releases through August 2024
- **Data quality:** 1 missing `track_album_name`; `track_album_release_date` stored as string with mixed formats (full date, year-only, one year-month); `track_id`/`id` are exact duplicates; `track_href`/`uri`/`analysis_url` are the same identifier in different formats

| Column (selected) | Type | Notes |
|---|---|---|
| `track_id` | string | Unique per track; use for dedup |
| `track_popularity` | int (0–100) | Target metric; min value in this dataset is 68 — see Limitations |
| `danceability`, `energy`, `valence`, `tempo`, `acousticness`, etc. | float | Spotify audio features, 0–1 scale (tempo in BPM) |
| `playlist_genre` / `playlist_subgenre` | string | Genre tag of the *playlist*, not the track itself |
| `track_album_release_date` | string → datetime | Cleaned in preprocessing |
| `track_artist` | string | Comma-separated for multi-artist tracks; exploded for artist-level analysis |

## Methodology

- **Deduplication first, always.** Every track-level statistic (medians, correlations, artist rollups) is computed on `df_unique` (one row per `track_id`), not the raw track-playlist grain — otherwise a track sitting in 6 playlists gets counted 6 times and skews any average toward whatever happens to be heavily playlisted.
- **ANOVA over eyeballing genre averages.** Different genres will always show *some* numeric difference in average energy; ANOVA + eta-squared tells us whether that difference is larger than we'd expect from sampling noise alone, rather than assuming any visible gap is meaningful.
- **KNN on acoustic features only, fit on the candidate pool.** For the sound-alike model, the nearest-neighbor index is fit on the *affordable* tracks (not the full catalog), and only on features that describe sound (danceability, energy, valence, tempo, acousticness) — deliberately excluding popularity and duration so the model can't "cheat" by just finding another popular track.
- **Random Forest + explicit imbalance check over trusting accuracy.** A classifier predicting rare "hit" outcomes can get 99%+ accuracy by predicting "not a hit" every time. We report precision/recall/support per class, not just accuracy, specifically to catch that failure mode.
- **OLS with audio-feature controls, not a raw month-vs-popularity comparison.** Isolating a "summer effect" requires holding energy/danceability/etc. constant — otherwise a seasonal pattern in *audio features* would masquerade as a seasonal pattern in popularity.

## Key Findings

**1. Hits are not more energetic — they're slightly less so.**
Comparing median audio features for tracks above vs. at/below the popularity-80 threshold: danceability −1.2%, energy −3.0%, valence −5.6%, tempo −0.06%. There is no "louder and faster wins" pattern in this data.

**2. Pop, latin, and rock dominate the top quartile, but pop is running away with it.**
Among the top 25% most popular tracks (popularity ≥ 79): pop accounts for 41.3%, latin 11.7%, rock 11.2%, gaming 10.7%, hip-hop 7.4% — the remaining dozen genres split the last ~28%.

**3. Genre doesn't explain energy differences in mega-hits (with an important caveat).**
ANOVA on tracks with popularity > 85 returns p = 0.80, eta-squared = 0.0009 — no detectable genre effect. But only 2 genres had enough mega-hits (≥5 tracks) to test, on a sample of 68 tracks total. This is a real finding, not a confident "universal formula" — it needs a larger, less-filtered sample to hold up.

**4. Tracks are getting shorter, but length doesn't drive popularity.**
Average duration fell from 3.8 minutes (2014) to 3.0 minutes (2023), yet the correlation between duration and popularity is essentially zero (r = −0.02). Marcus shouldn't pay a premium for longer tracks, and shouldn't reject shorter ones on length alone.

**5. The most "efficient" artists are not the most saturated ones.**
Ranking artists by average track popularity ÷ playlist appearances (fixed to average per unique track, not per track-playlist row — see Limitations), SZA tops the list at an efficiency score of ~79.8, appearing in only 1 playlist across 8 tracks, followed by Justin Timberlake and Kanye West. These are stronger direct-licensing candidates than ubiquitous, heavily-playlisted names.

**6. The sound-alike model works, but "affordable" here means "less mainstream," not "cheap."**
Matching Lady Gaga & Bruno Mars' "Die With A Smile" (popularity 100) against tracks scoring ≤75, the nearest acoustic match is "Siento que merezco más" by LATIN MAFIA (popularity 73, Euclidean distance 0.79 in standardized feature space) — a 27-point popularity gap with a near-identical audio profile. But this dataset's *floor* is popularity 68, so every "affordable" track is still a globally-charting song, not an unknown artist.

**7. Audio features alone cannot predict breakout hits (popularity ≥ 90).**
A Random Forest classifier hit 99.6% accuracy but 0% recall on the rare-hit class — it just learned to predict "not a hit" every time, because only 1 of 288 test tracks crossed that threshold. Lowering the bar to popularity ≥ 80 improved the sample size but recall was still just 3%. Audio features are not sufficient signal for breakout success.

**8. Release timing shows a real but modest effect, and it's broader than just "summer."**
Controlling for audio features, releases in most months (not only July–September) show a statistically significant popularity bump versus a January baseline, with August showing the largest effect. However, the model's R² is 0.07 — audio features and release month together explain only 7% of the variance in popularity. Timing matters at the margin; it is not the main driver.

## Recommendations

| Recommendation | Owner | Expected impact | Cost/risk |
|---|---|---|---|
| Don't hard-filter Aura Peak's algorithm on high energy/danceability | Elena, Content Strategy | Avoids curating a channel around a "loud = popular" assumption the data doesn't support | Low — reallocates existing curation criteria |
| Prioritize direct licensing outreach to high-efficiency, low-saturation artists (e.g. SZA, Justin Timberlake, Kanye West tier) | Marcus, Licensing | Access to reliably high-popularity catalog without bidding against every other buyer of the same 10 playlist-saturated names | Requires artist-by-artist negotiation; efficiency score is a shortlist, not a contract |
| Use the sound-alike model as a candidate generator, with mandatory human review before any client-facing swap | Marcus + Elena | Surfaces lower-cost substitutes with near-identical acoustic profiles | Model is genre/lyrics/culture-blind — brand-safety review is non-negotiable, not optional |
| Do not deploy a popularity-prediction filter in the current curation pipeline | Elena | Prevents shipping a classifier with ~3% recall on the exact outcome it's meant to catch | Needs a broader, non-pre-filtered Spotify sample before this is revisited |
| Treat late-summer release timing as a minor tiebreaker, not a strategy | Elena | Sets realistic expectations — a 7%-R² effect shouldn't anchor a launch calendar | None — this is a "stop overweighting" recommendation, not a spend |

## Limitations & Assumptions

- **This dataset only contains already-popular tracks** (`track_popularity` minimum is 68). There is no genuinely obscure or emerging artist in it, so any "cheap alternative" claim is relative — Marcus should treat this as "less mainstream," not "actually cheap," and cost data would need to come from a separate source.
- **No licensing cost data exists in this dataset.** All cost-saving language assumes popularity correlates with licensing fee, which is a reasonable proxy but unverified.
- **`track_popularity` is a single current snapshot**, not a time series — we can't tell if a track is rising, peaking, or declining, which matters for a "fresh content" strategy.
- **Playlist placement and popularity are correlated but we can't establish direction** — a track could be popular because it's playlisted, or playlisted because it's popular.
- **The genre-effect finding (ANOVA) rests on only 68 tracks across 2 genres** at the >85 popularity threshold — treat as directional, not conclusive.
- **To close these gaps we'd need:** time-series popularity history, actual licensing cost/royalty data by artist tier, and (ideally) skip/completion-rate data rather than an aggregate popularity score.

## Repo Guide

```
├── high_popularity_spotify_data.csv     # raw input data
├── Spotify_Data_Project.ipynb           # full analysis notebook (Q1–Q8)
├── aura_clean_dataset.csv               # deduplicated, cleaned track-level export
├── aura_ml_dataset.csv                  # scaled/encoded feature set used for modeling
├── requirements.txt                     # curated dependency list
└── README.md
```

**To reproduce:**
```bash
pip install -r requirements.txt
jupyter notebook Spotify_Data_Project.ipynb
```
Run all cells top to bottom — the notebook re-derives `df_unique` and `ml_df` from the raw CSV, so no manual setup is required beyond having the CSV in the same directory.

**requirements.txt (suggested, not auto-generated):**
```
pandas
numpy
scikit-learn
scipy
seaborn
matplotlib
statsmodels
```
