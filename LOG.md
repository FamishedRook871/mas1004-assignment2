# Log

One entry for each recipe, selection or Track B model you train, newest at
the bottom. The numbers come from `experiments.csv`. "Why" and "what it means"
are in your own words: your agent asks you for them and does not make them up.
End the log with your plan for what to try next.

<!-- The form of an entry:

## <date> <tag> (Track A or B)
- changed: what is different from the run before
- why: your reason
- validation: 0.000 (before: 0.000)
- what it means: what you think the result says
-->

## 2026-11-05 Step 1: Starter photo inspection (`look --every 20`)
- generated contact sheets in `results/curate/`:
  - `seoul`: 564 sampled photos on 12 sheets (`look_seoul_00` to `look_seoul_11`)
  - `busan`: 260 sampled photos on 6 sheets (`look_busan_00` to `look_busan_05`)
  - `tokyo`: 668 sampled photos on 14 sheets (`look_tokyo_00` to `look_tokyo_13`)
  - `osaka`: 360 sampled photos on 8 sheets (`look_osaka_00` to `look_osaka_07`)
  - `taipei`: 363 sampled photos on 8 sheets (`look_taipei_00` to `look_taipei_07`)
- observations (personal qualitative impressions of what tells the cities apart, not verified statistical findings):
  - **Seoul:** The photos look distinctly Korean, especially through Korean-language signage and familiar Korean urban surroundings. The cars include brands I would typically recognize from the US.
  - **Busan:** Shares Seoul's Korean visual identity, but the photos feel noticeably more coastal and ocean-oriented.
  - **Tokyo:** Has a distinctly Japanese atmosphere. The signage, vehicles, and surroundings feel recognizably Japanese, including Japanese car brands.
  - **Osaka:** Looks distinct in its own right while retaining a recognizable Japanese identity.
  - **Taipei:** Also has a distinct visual identity and does not look interchangeable with the Korean or Japanese cities.
  - **Overall observation:** Each city's photos seem to represent its respective country and location quite accurately. The differences appear to involve a combination of language, vehicles, surroundings, and environmental character, rather than just signage.

## 2026-11-08 baseline (Track A) — Step 2: Baseline training
- changed: unmodified starter recipe (`track_a/recipe.py`) trained on all 44,257 starter training photos (`seoul`: 11,274, `busan`: 5,191, `tokyo`: 13,347, `osaka`: 7,200, `taipei`: 7,245) for 249,984 photos (3,906 steps of batch size 64) with seeds 0 and 1.
- why: establish a baseline reference point for evaluating future single-variable experiments and measure how much validation score varies across random seeds alone.
- validation:
  - seed 0: `0.5023` (`seoul 0.5639`, `busan 0.3770`, `tokyo 0.6442`, `osaka 0.3534`, `taipei 0.5729`)
  - seed 1: `0.4907` (`seoul 0.5626`, `busan 0.3842`, `tokyo 0.6068`, `osaka 0.3811`, `taipei 0.5189`)
  - mean across seeds 0 and 1: `0.4965` (seed-to-seed difference: `0.0116`)
- per-city performance differences (factual observations):
  - Cities with more training photos achieve substantially higher validation recall: `tokyo` (`0.6442` / `0.6068`), `seoul` (`0.5639` / `0.5626`), and `taipei` (`0.5729` / `0.5189`).
  - Cities with fewer training photos in the same country have much lower validation recall: `busan` (`0.3770` / `0.3842`) and `osaka` (`0.3534` / `0.3811`).
- what it means: The baseline performance is relatively poor, with an average validation score of 0.4965. Tokyo, Seoul, and Taipei perform better than Busan and Osaka. This suggests that the model has difficulty distinguishing cities that share similar visual characteristics, although differences in training-image counts may also contribute.

## 2026-11-09 Step 3: Mistake analysis (`mistakes_baseline_seed0`)
- generated validation mistake sheets (`results/curate/mistakes_baseline_seed0_<city>_00.png` and `.csv`, 48 highest-confidence errors per city):
  - `seoul`: 1,063 of 2,435 wrong
  - `busan`: 866 of 1,390 wrong
  - `tokyo`: 1,380 of 3,881 wrong
  - `osaka`: 1,374 of 2,128 wrong
  - `taipei`: 958 of 2,243 wrong
- largest confusion patterns in the full validation confusion matrix (`results/baseline_seed0/result.json`):
  - **Osaka → Tokyo:** 840 of 2,128 Osaka validation images (39.5%) are predicted as Tokyo (compared to 752 predicted correctly as Osaka); 38 of the top 48 Osaka mistakes are predicted as Tokyo.
  - **Busan → Seoul:** 446 of 1,390 Busan validation images (32.1%) are predicted as Seoul (compared to 524 predicted correctly as Busan); 35 of the top 48 Busan mistakes are predicted as Seoul.
- recurring visual patterns supported by the mistake sheets and CSV analysis:
  1. **Shared country-level characteristics:** Architecture, language/signage, and vehicles often look similar between cities in the same country (e.g., Korean police/post office signage and temples in Busan predicted as Seoul; Japanese residential streets and shrines in Osaka predicted as Tokyo).
  2. **Images with little geographic context:** High-confidence mistakes include close-ups of cars/showrooms, airplanes on airport tarmacs, food plates, and indoor corridors/rooms where street-level city cues are absent.
  3. **Repeated frames from the same YouTube videos:** Multiple top-48 mistakes per city originate from the same nighttime, rainy, or dashcam walking/driving video sequences (e.g., 10 frames from `youtube_A6PuVf6KcnA` in Tokyo, 10 from `youtube_kCaJ_4NFmJ0` and 8 from `youtube_mCj8vSb6Iow` in Busan, 7 from `youtube_-yRGrGnYB50` in Taipei).
- student's own summary of two commonalities: Two major commonalities are the similarities between cities within the same country and the lack of useful geographic information in certain images. Korean signage, architecture, and vehicles can appear across both Seoul and Busan, while Japanese visual characteristics are shared between Tokyo and Osaka. Additionally, close-up objects and indoor images often lack recognizable city-specific clues. Repeated frames from the same videos may also contribute disproportionately to the mistakes.
- hypotheses / potential directions to test next (not yet verified):
  - **Class-balanced sampling (`sample_weights`):** Test whether drawing each city equally often reduces majority-class attraction (`Osaka → Tokyo` and `Busan → Seoul`).
  - **Reviewing horizontal flipping (`RandomHorizontalFlip`):** Test whether removing left-right flipping preserves driving-side and script/signage orientation cues.
  - **Adjusting random crop scale or investigating training-image quality:** Test whether a less aggressive crop scale or curating out non-street/indoor training photos preserves geographic context.

## 2026-11-09 balanced (Track A)
- changed: in `track_a/recipe.py`, implemented `sample_weights(cities, paths)` using `collections.Counter(cities)` to return `[1.0 / counts[c] for c in cities]`, so all five cities are drawn equally often during training (no other changes).
- why: the baseline training set has substantial class imbalance (`tokyo` 13,347 and `seoul` 11,274 vs. `busan` 5,191, `osaka` 7,200, and `taipei` 7,245); test whether equal city sampling improves recognition of underrepresented cities, particularly Busan and Osaka.
- validation: `0.5071` on seed 0 (before: `0.5023` on baseline seed 0; `0.4965` baseline mean across seeds 0 and 1)
  - per-city validation scores (`balanced` seed 0 vs. `baseline` seed 0):
    - `seoul`: `0.4969` (before: `0.5639`, `-0.0670`)
    - `busan`: `0.4388` (before: `0.3770`, `+0.0618`)
    - `tokyo`: `0.5202` (before: `0.6442`, `-0.1240`)
    - `osaka`: `0.4605` (before: `0.3534`, `+0.1071`)
    - `taipei`: `0.6188` (before: `0.5729`, `+0.0459`)
  - confusion matrix highlights: `Osaka → Tokyo` errors dropped from 840 to 526 (Osaka correct rose from 752 to 980), and `Busan → Seoul` errors dropped from 446 to 378 (Busan correct rose from 524 to 610).
- what it means: Busan (`+0.0618`) and Osaka (`+0.1071`) improved substantially, and Taipei (`+0.0459`) also gained, while the previously over-predicted majority classes Seoul (`-0.0670`) and Tokyo (`-0.1240`) declined. Overall macro validation accuracy increased from `0.5023` (seed 0) and `0.4965` (baseline two-seed mean) to `0.5071`, so we retain the `balanced` sampling change. A limitation to note is that `balanced` has only been evaluated on one seed (`seed 0`), and the `+0.0048` gain over baseline seed 0 (`+0.0106` over the baseline mean) is within the `0.0116` seed-to-seed variation observed on the baseline.



