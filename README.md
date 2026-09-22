# True Pass — Detecting Goal-Pattern Creating Passes

**Status: design stage (v1, September 2026). No implementation yet — this repository holds the plan.**

Not every pass before a shot is a meaningful one. A defender-to-defender ball in the build-up is
counted the same as the through ball that breaks the defensive line. True Pass is a method for
telling them apart: it scores each pass by what it does to the defence, and then judges a *new*
pass by finding the most similar situations that came before it.

Full plan (diagram): [Excalidraw board](https://excalidraw.com/#json=cl-oehe1NstMuwYiW3T0W,uHVbXlCW1emSwhSSYRBIbA)
· source file in this repo: [`true_pass_plan.excalidraw`](true_pass_plan.excalidraw)

Data: [Hudl StatsBomb Open Data](https://github.com/hudl/open-data) — event data joined with 360
freeze frames.

---

## The problem with 360 data

StatsBomb 360 gives a freeze frame for each event: the positions of every player the camera can
see at that moment. It is the closest thing to tracking data in an open dataset, but it has two
gaps that this plan has to work around.

**Only one player is labelled.** In a frame, the event player carries `actor = true`. Everyone
else is an anonymous point with nothing but a teammate/keeper flag — no name, no shirt number, no
identity across frames.

**The camera does not see everyone.** Players outside the broadcast view simply do not appear, so
the receiver of a pass, or the defender who was beaten by it, may be missing from the very frame
where they matter most.

Everything in phase 2 exists to work around these two gaps.

---

## 1. Data

Matches with 360 data are kept (men's matches only in v1). Event data and 360 frames are joined on
`event_uuid`: the event tells us **who** (passer = event player, receiver = `pass.recipient`), the
frame tells us **where** (all visible positions).

An **attack** is one StatsBomb possession that ends with a shot. Excluded: own goals, cancelled
goals and penalties. Attacks that start from a set piece are kept but flagged with `play_pattern`.
Possessions with fewer than two completed passes are dropped — there is no chain to analyse.

Possessions that do **not** end with a shot are collected as a **control group**, not as negative
examples. A pass can be excellent and still be followed by a miscontrol or a wasted shot, so
treating those passes as "bad" would teach the wrong lesson. The control group is used for
validation instead (phase 3).

**Pass pool.** Every completed pass by the attacking team inside the possession, before the shot.
Incomplete passes are excluded — with no receiver, receiver-based metrics cannot be computed.
Passes with no 360 frame are dropped. Set-piece deliveries (corner, free kick, throw-in, goal
kick) stay in the pool, flagged with `pass.type`, and are only ever compared against passes of the
same type.

## 2. Freeze frames

**Coordinates and grid.** StatsBomb yards (120 x 80) are converted to meters (109.7 x 73.2 m);
attacking direction is already normalised left to right. The pitch is sliced into N x N m cells
(N = 2 to start), giving a 55 x 37 grid, with `cell = (floor(x / N), floor(y / N))`.

**Player continuity.** Between consecutive 360 frames, points of the same team are matched with
the **Hungarian algorithm**, using distance as the cost. Any pairing that would require more than
`v_hard x dt` of movement is rejected as physically impossible; unmatched points mean a player
entered or left the camera view. Identity anchors come from the event stream: the receiver is
labelled in frames where he is the actor (his Ball Receipt or Carry events), defenders in their
own defensive events. The goal is **continuity, not full identity** — knowing that this point is
the same person as that point one frame earlier is enough.

**Missing players — reachable areas.** When a player who matters is not in the pass frame, their
position is not guessed as a single point but as a probability mass over the grid, bounded by what
a human body can do:

| | Case 1: receiver missing at pass time | Case 2: defender missing at pass time |
|---|---|---|
| Known | Pass receipt location, pass duration, last frame where the receiver was seen, the gap `dt1` to the pass frame | Defender's position at receipt time (via frame matching), last frame where he was seen |
| Reachable area | `disk(last seen, v x dt1)` ∩ `disk(receipt location, v x pass duration)` | `disk(receipt-time location, v x pass duration)` ∩ `disk(last seen, v x dt)` when seen before |

The intersection is spread over grid cells as a probability mass. If it is **empty even with
`v_hard`**, the position is mathematically unreachable and the plan falls back to a single known
point: the pass receipt location (case 1) or the defender's receipt-time location (case 2).

**Physical limits** come from the literature, not from guesses:

- typical elite top speed `v` = 9.1 m/s (32.9 ± 1.4 km/h) — used to build reachable areas
- hard limit `v_hard` = 10.4 m/s (37.38 km/h, highest recorded) — used to reject matches and
  declare unreachability
- max in-match acceleration 5.35 m/s², theoretical range 4.5 – 8.68 m/s²

Sources: [Velocity and Acceleration Profile in Football (2025), Sports/MDPI](https://www.mdpi.com/2075-4663/13/1/18)
· [Analyzing Soccer Match Sprint Distances (2024), PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC11167479/)

## 3. Pass filtering — confidence score

Each pass gets a score built from metrics that describe what the pass did to the defence. Every
metric is normalised to 0 – 1.

1. **Players taken out of the game.** Opponents who were goal-side of the ball before the pass and
   are no longer goal-side of the receiver after it. The search area is a half-circle centred on
   the passer, radius equal to the pass length, oriented in the pass direction. More players
   removed, higher score.
2. **Opponents between receiver and goal, and shot angle.** Opponents in the receiver-to-goal
   corridor counted before versus after the pass; fewer after is positive. Shot angle is the angle
   subtended by the two goal posts from the receiver's position.
3. **Progression.** The reduction in distance to goal, comparing ball location at the pass with
   the location at receipt. Distance rather than x-advance, so that a wide-to-central pass is
   valued properly. This balances metrics 1 and 2.
4. **Free space around the receiver.** Opponents within r = 5 m of the receiver: in space, or
   tightly marked.
5. **Diagonal passes** (a modifier, never a metric on its own). A pass at 20° – 70° to the goal
   axis is diagonal; lateral displacement above 20 m is flagged as a switch of play. This changes
   the weights of the other metrics rather than adding to the score.

*Candidate metric, to be tested:* goalkeeper position, which freeze-frame xG models use (keeper
distance to the goal line or to the shooter). For passes it may only matter in the final third.

**Aggregation.** `metric score` is a weighted sum of metrics 1 – 4 (equal weights to start),
adjusted by metric 5, and is **independent of how the attack ended**. A missing-data multiplier
lowers confidence when many of the players involved were predicted (heat area or fallback) rather
than seen — typically mid-pitch passes where the camera shows little.

**xG bonus (the only outcome-based component).**

```
confidence = metric score + xG bonus
xG bonus   = shot xG x decay ^ (actions between the pass and the shot)
```

An attack that ended in a better chance raises the confidence of the passes that built it, and
earlier passes in the chain get less credit — borrowing VAEP's idea that only the next ~10 actions
matter. **No shot means a bonus of zero, never a penalty:** a good pass that led nowhere keeps its
metric score.

**Chain check.** Can this pass create another true pass? Using the same similarity machinery:
within 5 s of receiving, can the receiver produce a high-confidence pass? Whether this acts as a
bonus or a hard filter is still open.

**Validation without labels.** There is no ground truth for "true pass", so the metrics are
validated by distribution: compare metric scores of passes in shot-ending attacks against those in
the control group. If high scores appear more often before shots — and before higher-xG shots —
the metrics are measuring something real. The same comparison tunes the metric weights and the
true-pass threshold.

## 4. Pass vector

Each pass becomes a fixed-size, multi-channel tensor over the grid, so that the representation
does not depend on how many players happen to be visible:

| Channel | Contents |
|---|---|
| 1 | Attacking team positions |
| 2 | Defending team positions |
| 3 | Passer location |
| 4 | Receiver location / pass end location |

Players who were seen are drawn as Gaussian blobs (σ = 2 – 3 m), so that a one-cell shift does not
destroy similarity; players who were predicted are drawn as their reachable-area probability mass.
Channels stay separate so they can be weighted independently at comparison time.

Metadata is kept **outside** the tensor: confidence score and its per-metric breakdown, xG bonus,
`pass.type`, start and end zone, `match_id` and event ids.

## 5. Similarity phase

Database and test data are split **by match**, so no pass from a test match can appear among its
own neighbours. A new pass runs through the same pipeline and becomes a tensor plus metadata.

1. **Metadata pre-filter.** Keep only candidates with the same `pass.type` (open play, or the same
   set-piece type) and matching start/end pitch zones — the metadata filter of a RAG system.
2. **Retriever: cosine.** Cosine similarity computed **per channel**, then combined as a weighted
   sum, so the ten-player team channels cannot drown out the passer and receiver channels. Cells
   near the ball are weighted more heavily than distant ones. Result: top 50. Brute force is
   enough at this scale (tens of thousands of passes, no vector index needed).
3. **Re-ranker: Wasserstein.** The 50 candidates are re-ranked by Wasserstein (optimal transport)
   distance, which compares player distributions by how much movement it would take to turn one
   into the other, and is robust to small positional shifts. Result: top K = 10.
   (Role-alignment methods such as Chalkboarding need tracking identities, which 360 data does not
   provide.)
4. **Decision: weighted kNN.**

   ```
   prediction = Σ (sim_i x confidence_i) / Σ sim_i      over the top K
   ```

   If every neighbour falls below the minimum similarity threshold, the answer is **"no
   precedent"** rather than a forced number. Evaluation follows xG-model practice on held-out
   matches: Brier score, AUC and calibration (reliability diagram).

## Planned stack

| Purpose | Tools |
|---|---|
| Environment | Python 3.12 + Jupyter |
| Data | `statsbombpy`, `pandas`, `pyarrow` (event + 360 join, stored as Parquet) |
| Action chain | `socceraction` (SPADL) — standard action format, actions-until-shot for the xG decay, VAEP |
| Geometry | `shapely` — disk intersections, half-circle search area |
| Matching | `scipy.optimize.linear_sum_assignment` — Hungarian algorithm |
| Tensor | `numpy`, `scipy.ndimage.gaussian_filter` |
| Similarity | `scikit-learn` (cosine), `POT` — Python Optimal Transport (Wasserstein) |
| Evaluation | `scikit-learn.metrics` — Brier score, ROC-AUC, `calibration_curve` |
| Visualisation | `mplsoccer` — pitch plots, reachable areas, similar passes side by side |

Not needed at this scale: FAISS or a vector database (brute force is enough), and a deep learning
framework (only if learned embeddings are added later).

## Tunable parameters

| Parameter | Starting value |
|---|---|
| Grid cell size N | 2 m |
| Gaussian σ | 2 – 3 m |
| Free-space radius r | 5 m |
| Diagonal band | 20° – 70° |
| Switch of play | lateral > 20 m |
| Chain window | 5 s |
| Retriever top / final K | 50 / 10 |
| Minimum similarity threshold | TBD |
| xG decay / horizon | TBD / ~10 actions |
| v / v_hard | 9.1 / 10.4 m/s |

## Open questions

- Metric weights and the true-pass threshold (to be tuned with the control-group validation)
- Chain check: bonus or hard filter?
- Decay factor of the xG bonus
- Channel weights in the cosine stage
- Goalkeeper position: worth its own metric?

## License

Documentation and diagrams: CC BY-NC-SA 4.0. Source code (when added): PolyForm Noncommercial
1.0.0. Commercial use requires a separate agreement — see [LICENSE.md](LICENSE.md).

---

Yusuf Erdem Altinsoy · [GitHub](https://github.com/erdemalti0) ·
[Kaggle](https://www.kaggle.com/yusuferdemaltinsoy)
