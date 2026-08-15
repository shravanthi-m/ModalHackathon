# Next direction — couple wrist motion with body coordinates (multimodal)

**Status: PARKED here after determining wrist-only can't do it. Everything current is
pushed. Resume from this doc.**

## What we determined (so nobody re-runs it)
1. Impulse detector flags forceful **intentional** motion, not failure — video-validated:
   precision@10 = 1/10 (original), 1/5 (after B's recalibration). Same false positives:
   soap squirt, garbage disposal, filling a bowl.
2. **Tested the impulse+correction fix** on 15 human-labeled clips → precision **2/13**,
   no lift. The "correction" wiggle after an impulse is kinematically identical to normal
   busy hand motion in cluttered tasks (nearly every impulse showed ≥3 speed reversals
   within 1.5s, real or not). **→ wrist trajectory alone has a real ceiling.**

## Why multimodal is the way forward
A genuine drop/slip leaves a signature **beyond the wrist** that intentional forceful
motion does not:
- **Gaze / head shift** — the person looks at the dropped object (`obs_head_pose`).
- **Hand opening** — the grasp releases (full MANO keypoints, not just joint 0).
A soap squirt or disposal dump has the wrist impulse **without** the gaze snap + hand-open.

## Channels already in every episode's zarr (unused so far)
- `obs_head_pose` (7) — head/gaze proxy
- `left/right.obs_keypoints` (63 = 21×3 MANO) — full hand pose / **aperture** (we only used joint 0)
- `left/right.obs_ee_pose`, `left/right.obs_wrist_pose` (7)

## Proposed gate for whoever picks this up
`failure = wrist impulse  AND  (head-pose angular-jerk spike  OR  hand-aperture jump)  within ~1s`
- head angular velocity: quaternion deltas of `obs_head_pose`
- hand aperture: spread/extent of the 21 MANO keypoints; a release = sudden increase
- validate the same way we did: watch the new top-10, count real failures

## UPDATE — multimodal tested, also no lift (2026 hackathon)
Prototyped the gate above (`scratchpad/test_multimodal.py`): wrist impulse AND a nearby
head-angular-jerk **or** hand-aperture spike, on the same 15 labeled clips.
**Result: precision 2/13 — identical to wrist-only and to impulse+correction.**
Head jerk and hand-aperture spikes fire near ~every impulse because dishwashing scenes
are constantly active in *every* channel. Three approaches now tested, all ~2/13:
wrist-impulse · impulse+correction · wrist+head+hand.

**Conclusion:** the failure event (an object visibly falling) is not cleanly present in the
available motion/pose channels for cluttered tasks — it's semantic/visual. Deterministic
kinematic detection has a real ceiling here. This is the honest, defensible result.
Multimodal *done carefully* (release-specific hand-open, gaze-locked-to-object) is real
future work, not a hackathon-timeframe win.

## UPDATE 2 — careful multimodal tested too; the real blocker is DATA, not features
Refined gate (`scratchpad/test_multimodal_v2.py`): impulse + **sustained hand release**
AND **gaze redirect** (head turns to a new heading), sequential/specific. Result:
flagged 1 (a false positive), **caught 0/2 real** — the strict signature isn't even
present in our 2 confirmed drops. Four feature families now tested, none separate:
`wrist 2/15 · correction 2/13 · naive-multimodal 2/13 · careful-multimodal 0/1`.

**Root cause:** we have **2 confirmed failures** out of ~15 watched. You cannot tune or
validate a detector on 2 positives — every feature result is anecdotal. The wall is data,
not cleverness.

## Requirements to continue building this productively
1. **A labeled failure set (blocker #1).** ~30–50 confirmed failure clips + matched
   negatives, from a labeling push (watch N random wash_dishes clips) or failure-annotated
   episodes. Nothing below is buildable or trustworthy without this.
2. **An object / contact signal (blocker #2).** The true discriminator — *object leaves the
   hand* — is NOT in wrist/head/hand pose. Get it from either (a) the RGB frames + a vision
   model (object separates from hand / enters sink), or (b) a grasp/contact-state channel.
   Pose is a proxy that doesn't carry this event.
3. **Refined pose features, tuned on the labeled set:** release-specific aperture (hand
   stays empty), gaze redirect-and-fixate, impulse→release→gaze ordering.
4. **Held-out validation:** precision@k on video, on episodes NOT used for tuning.
5. **Scale on Modal** once validated — the audit harness already exists (`audit_modal.py`).

**If we don't finish:** the submission is the rigorous ceiling result (4 approaches, video-
validated) + this concrete roadmap. That's a stronger, more honest Track-3 answer than a
number that doesn't survive a judge watching one clip.

## Artifacts to resume from
- Labeled clips + verdicts: `~/Documents/Hackathon/clips/`, `clips_recal/` (+ scorecards)
- wash_dishes scores: `audit_washdishes.parquet`, `audit_wd_recal.parquet`
- Correction-gate test (the negative result): `scratchpad/test_correction.py`
- Findings: `VALIDATION_FINDINGS.md`  ·  Slide: `SLIDE.md`
