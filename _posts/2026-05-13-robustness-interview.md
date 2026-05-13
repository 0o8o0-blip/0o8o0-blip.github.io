---
layout: post
title: "How much data do you need to make an ACT policy robust? A field report."
date: 2026-05-13
slug: robustness-interview
image: /assets/figures/fig7_camera_frames_clean_vs_collapse.png
---

![Wrist-camera frames: clean run vs. collapse](/assets/figures/fig7_camera_frames_clean_vs_collapse.png)
*Wrist-camera frames from a clean run (top) and a collapse run (bottom) — what "robust" and "not robust" actually look like from the policy's point of view.*

### Intro 

So last week I published [auto_soarm](https://github.com/0o8o0-blip/auto_soarm), an auto research loop using the soarm101 and a tablet for feedback on robot tasks. It has a few simple games at the moment. 
The main one is just tap a circle. Over the last week I let agents explore the ACT robustness perturbations. I mostly left the agent alone this week, other than the occasional "you are doing great keep going".

Three perturbation axes, evaluated independently and combined:

1. **Lighting** — tablet backlight (`brightness 10-250`) and uncontrolled room ambient (which varies across 24 hrs).
2. **Pose** — initial home pose perturbed by `home_shift_pan/lift/wf` ticks before each episode. The arm starts the episode "off-pose" and has to recover.
3. **Time of day** — same setup, different ambient lighting depending on hour. The arm is placed beside a window so light changes all the time. 

The success metric is hits / 12 episodes at each eval condition. 

### What was the baseline before you started pushing on this?

Brittle. The original `rainbow5_v1_balanced` was 20/20 in the lighting and pose it was collected in, and dropped to ~0/12 in any noticeably different condition. 


### What was the first intervention?

Random shoulder pan of 15 ticks. 

The first model trained with these (60 eps, `home-rand pan 15`) — call it v4 — survived ±75 pan-shift evals in the evening it was trained in. 
That was the first signal that randomization alone could move the needle.

### Then what broke?

Overnight and the next morning. v4 was collected in the evening; at dawn it collapsed to 4/12 only at b250 (and 0/12 at lower brightness). A separately-collected dawn-only model (v9) collapsed at 1 PM. 
A 4-AM-only model (v8) was dead by 8 AM.

So single-time-of-day collects were structurally incapable of crossing the lighting day. The randomization helped within a band but couldn't extrapolate.

### How did you fix the time-of-day problem?

By collecting demos at multiple times of day and merging them. The current SOTA `v9v10v13v15` is four collects:
- v9: dawn (~6:30 AM)
- v10: late morning (~10:20 AM)
- v13: night (~11 PM), with wider home-rand
- v15: night (different night), with even wider home-rand (`pan 50 lift 25 wf 30`) post-recalibration

Merged: 169 episodes. Trained dense to 15,000 steps. That model holds 7-11/12 at b128 across the 6 PM → 6 AM window.

![Diurnal robustness over the multi-collect ladder](/assets/figures/fig3_diurnal_curves.png)
*Each added time band extends the robustness envelope. The current SOTA (green) holds through the night but degrades in the unfilled 10:30 AM – 2 PM window — that's the story the rest of this writeup is partly chasing.*

### Why three time bands and not two? What's the failure mode of just two?

Two bands (v9+v10 at 10k steps) actually held longer in the evening than I expected — 6-7/12 deep into the night — but eventually collapsed at midnight. The model interpolates between bands it has seen, but 
once the ambient drifts past the latest band it's seen, it falls off. A third band on the far side of the day fixes that.

There's also a counterintuitive finding: a noisy four-band merge — one that pulled in two late-afternoon collects with ~25-30% reject rate in the source data — actually overfit and *collapsed earlier* than the two-band morning+dawn baseline. 
Adding more data doesn't help if the added data is noisier than what's already there.

### Why is Pose perturbation harder than lighting?

Pose perturbation is harder because **pan/lift/wf aren't just kinematic changes, they're perceptual confounds.** The camera is wrist-mounted. 
So a different pan doesn't just mean different joint angles — it means the camera sees the tablet at a different angle, with different perspective distortion, different reflections off the screen, 
different background pixels visible. The encoder has to learn pan-conditional perception, not just pan-conditional motion.

I noticed this when running combined-axis evals on the diurnally-robust SOTA. Each axis alone was fine:
- pan -25 alone: 8/12
- lift +15 alone: 12/12
- wf +15 alone: 11/12

But combined?

- pan -25 + lift +15 + wf +15: **0/12, catastrophic collapse.**

The three-axis combination wasn't anywhere in the training distribution because home-rand applies each axis independently per episode and you basically never get all three at their extremes simultaneously by chance. So the joint envelope had a hole.

### How'd you close that?

Added a fifth collect with wider home-rand (`pan 50 / lift 25 / wf 30`, up from `pan 30 / lift 20 / wf 20` on v13). The same 3-axis perturbation went from 0/12 to **12/12 perfect** on the new SOTA. 
Individual axes also extended significantly:

- lift cliff: ±15 → ±60 (lift +60 went from 0/12 to 12/12)
- pan negative cliff: ±50 → ±75 to ±100
- wf cliff: ±30 → ±60

The pose envelope after that collect is genuinely large. Not unbounded — pan +X is still weak due to a residual calibration asymmetry on the left side of the tablet — but the catastrophic-collapse regime is gone.

### Why does wider home-rand need more episodes?

Two reasons, one obvious and one that took an experiment to see.

**The obvious one is volume.** When I tried to go even wider (v16: `pan 75 / lift 40 / wf 50`), the pose volume jumped 4×:

| | pan range | lift range | wf range | volume (units) |
|---|---|---|---|---|
| v15 home-rand | 100 | 50 | 60 | 300k |
| v16 home-rand | 150 | 80 | 100 | 1.2M |

So the practical rule isn't "wider rand needs more data" — it's "each new randomized axis multiplies the state-space volume, and you need enough samples per cell that random uniform sampling at the edges averages out." With N=60 over three axes, it pretty obviously doesn't.

### Can you put a number on it? How many samples for "robust"?

It depends on the envelope you want. From the actual numbers I have:

| Envelope | Estimated dataset | Collects |
|---|---|---|
| Single-condition (no robustness) | 60 eps | 1 |
| Robust within one time band, pose ±15 | 60 eps | 1 (v4 recipe) |
| Robust 6 PM → 6 AM at current pose envelope | 169 eps | 4 (current SOTA) |
| Robust full 24 hours at current pose envelope | *open* — initially estimated ~250-300 eps, but a 249-eps merge with a mid-day collect regressed at both 15k and 25k steps; see "did the prediction hold up?" |
| Robust against day-to-day environmental drift (different weather, recalibration cycles) | ~450-500 eps | 8-10 (multi-day re-collects) |
| Robust at v16's pose ambition (pan ±75) | ~500-700 eps | 10+ |

These are *my numbers on my bench*. The shape probably generalizes; the absolute counts almost certainly don't.

The bigger lesson is that the marginal value of more episodes is heavily nonlinear:
- 60 → 143 eps (adding the third time band): huge jump
- 143 → 169 (adding wider pose-rand): good
- 169 → 219 (more wider pose-rand): **net negative**

Once you have the bands, the ROI on more episodes within existing bands is flat. The next axis you haven't covered is worth more than density in axes you already cover.

![Episode count vs aggregate diurnal score](/assets/figures/fig5_episode_count_vs_robustness.png)
*Aggregate hits/36 across three b128 time bands. The trajectory climbs steeply with the first three time-band additions, peaks at the 4-collect SOTA, then drops sharply on the 5-collect wider-rand merge — the v16 cautionary tale.*


### Where would more data stop helping?

My best guess from extrapolating the curve is somewhere around 1000 episodes for this task with ACT specifically — but I haven't actually run past 250, so treat that as informed extrapolation rather than measurement. Past that I'd expect to hit a policy-class ceiling rather than a data ceiling — the residual failures will be perceptual (camera glare, unusual lighting) that an MLP-on-encoded-features model can't really learn its way out of without a much bigger visual backbone.

That's where I'd consider switching to a VLA with a pretrained encoder instead of pushing ACT data harder. I've tried SmolVLA fine-tunes at 23 and 52 episodes and both failed to learn the task at all, so I know the scale floor is higher than that. With 169-250 eps of well-randomized data and a flat LR, a VLA fine-tune is a natural next experiment if the goal is generalization rather than in-distribution accuracy.


### What's the one-paragraph summary?

ACT policies trained on a single condition are brittle in obvious ways. Adding pose and brightness randomization at collect time fixes brittleness within a condition. Adding *multiple time-of-day collects* fixes brittleness across the diurnal cycle. Each randomization axis multiplies the state-space volume, so wider randomization needs more data — but with sharply diminishing returns, and a real risk that random-sampling variance at the edges makes things worse, not just thinner. The practical recipe that worked on my bench gets you to `~60 episodes × ~3 time bands × wider-rand on at least one of them × 15k training steps`, and from there to a model that holds 7-11/12 through the 6 PM → 6 AM window. Past that point, the work stops being a recipe and starts being a research problem — both attempts I made to extend the SOTA beyond 169 episodes regressed it, even when the added data was clean and training was scaled with dataset size. The next interventions worth trying are stratified sampling on the collect, time-conditioning on the policy, or switching policy class to a pretrained-encoder VLA.
