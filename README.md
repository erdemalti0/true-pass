# True Pass — Detecting Goal-Pattern Creating Passes

**Status: design stage (v1, September 2026). No implementation yet — this repository holds the plan.**

Not every pass before a shot is a meaningful one. A defender-to-defender ball in the build-up
counts the same as the through ball that breaks the defensive line. True Pass is a method for
telling them apart: it scores each pass on what it does to the defence, and then predicts whether
a *new* pass is a true pass by finding the most similar situations that came before it.

Full plan (diagram): [Excalidraw board](https://excalidraw.com/#json=cl-oehe1NstMuwYiW3T0W,uHVbXlCW1emSwhSSYRBIbA)
· source file in this repo: [`true_pass_plan.excalidraw`](true_pass_plan.excalidraw)

Data: [Hudl StatsBomb Open Data](https://github.com/hudl/open-data) — event data joined with 360
freeze frames.

## Pipeline

**1. Data.** Keep matches that have 360 data. An *attack* is one StatsBomb possession that ends
with a shot; own goals, cancelled goals, penalties and possessions with fewer than two completed
passes are dropped. Possessions that do not end with a shot are kept as a control group, not as
negative examples — a pass can be good even when no shot follows.

**2. Freeze frames.** Convert the pitch to meters and slice it into an N x N m grid (N = 2 to
start). Event data says *who* passed to whom; the 360 frame says *where* everyone stood, but only
the event player is labelled. Players are followed between consecutive frames with the Hungarian
algorithm, rejecting any pairing that would need more movement than a player can physically
produce. When a player is missing from a frame, their possible location is a reachable area —
the intersection of two disks built from the last time they were seen and from where they must
have been at pass receipt — spread over the grid as a probability mass.

Physical limits come from the literature: 9.1 m/s for typical elite top speed, 10.4 m/s as a hard
limit ([Sports, MDPI 2025](https://www.mdpi.com/2075-4663/13/1/18),
[PMC 2024](https://pmc.ncbi.nlm.nih.gov/articles/PMC11167479/)).

**3. Confidence score.** Each pass is scored on how many opponents it takes out of the game, how
many defenders remain between receiver and goal, how much it reduces the distance to goal, and how
much space the receiver has; diagonal passes and switches of play modify those weights. An xG
bonus, decayed by the number of actions between the pass and the shot, is added on top. Passes
with a large share of predicted (rather than seen) players get a lower confidence multiplier.
Since there are no labels, the metrics are validated by comparing their distribution in
shot-ending attacks against the control group.

**4. Pass vector.** Every pass becomes a fixed-size, multi-channel grid tensor — attacking team,
defending team, passer, receiver — with seen players drawn as Gaussian blobs and predicted players
as their probability mass. The confidence score and pass metadata stay outside the tensor.

**5. Similarity.** For a new pass: filter candidates by metadata (pass type, start and end zone),
retrieve the top 50 by per-channel weighted cosine similarity, re-rank them to the top 10 with
Wasserstein (optimal transport) distance, and predict with a weighted kNN over their confidence
scores. If every neighbour falls below a minimum similarity, the answer is "no precedent" rather
than a forced number. Database and test data are split by match to avoid leakage, and results are
evaluated the way xG models are: Brier score, AUC and calibration curves.

## Planned stack

Python 3.12 · statsbombpy, pandas, pyarrow · socceraction (SPADL) · shapely ·
scipy.optimize.linear_sum_assignment · numpy, scipy.ndimage · scikit-learn · POT (Python Optimal
Transport) · mplsoccer

## Open questions

Metric weights and the true-pass threshold, whether the chain check is a bonus or a hard filter,
the xG decay factor, channel weights in the cosine stage, and whether goalkeeper position deserves
its own metric.

## License

Documentation and diagrams: CC BY-NC-SA 4.0. Source code (when added): PolyForm Noncommercial
1.0.0. Commercial use requires a separate agreement — see [LICENSE.md](LICENSE.md).

---

Yusuf Erdem Altinsoy · [GitHub](https://github.com/erdemalti0) ·
[Kaggle](https://www.kaggle.com/yusuferdemaltinsoy)
