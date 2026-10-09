---
layout: post
title: "The Probe That Passed Before Training Began"
date: 2026-10-9
image: /assets/img/probe-controls.png
image_alt: "Held-out probe error at each site in a small transformer, for two trained models and for an untrained model with the same starting weights. The untrained model is never far behind."
image_caption: "Held-out probe error at each site in the network, for two trained models and for an untrained model with the same initial weights. Lower is better. The dotted line is a probe fitted on the raw inputs alone. Panels a and b of Figure 7 from the paper."
---

Early in my orbit study, a probe handed me the number 0.98, and it was wrong in a way that is easy to miss.

The study trains small transformers to predict where a planet will be next, and then asks what the model learned along the way. The standard tool for the second part is a linear probe: a regression from the activations inside the network to some quantity you care about, such as the shape of the orbit. If the regression succeeds, the reasoning goes, the model must contain that quantity. [Liu and colleagues at Stanford](https://arxiv.org/abs/2602.06923) used probes this way to show that a transformer with a long context learns the ellipse, as Kepler would have, while a transformer restricted to its last two positions learns the force, as Newton would have. I reproduced their result with their own code before changing anything, and the transition appeared exactly where they said it would.

Then I ran their probe on a model that had taken a single optimizer step. It scored 0.98 on the Keplerian variables.

One step is, for any practical purpose, no training at all. The weights sat a hair away from their random starting values. Yet by the metric in common use, this model already knew the shape of the orbit almost perfectly. Either transformers learn orbital mechanics faster than any model has learned anything, or the probe was measuring something other than what the model had learned.

## What a probe can do on its own

A linear probe is a model in its own right. It has parameters, and given enough freedom it will find structure that the network never used. Two features of the standard setup gave it that freedom. The probe was fitted and scored on the same data, so a regression that fitted noise was rewarded for it. And the metric took the best score across thirteen sites in the network, so whichever site happened to give the probe the easiest job was the one that counted. Best of thirteen, on in-sample fit, is a recipe for a high number.

The raw material helped too. The first layer of a transformer is a linear map of the input positions plus a position embedding. Even in an untrained network, the activations are a lightly scrambled copy of the positions the model has seen, and the orbital elements are a smooth function of those positions. A regression with any capacity at all can recover a good deal of that from almost any reasonable transformation of the inputs. The probe was doing the work and the model was taking the credit.

None of this is new as a warning. Hewitt and Liang proposed control tasks for exactly this reason, and Belinkov's review of probing lists the same failure modes. What was new to me was seeing it produce a number as clean as 0.98 on a model that had learned nothing, in a setting where I had been ready to believe it.

## Three controls

I rebuilt the probes before running the real experiment. Every model was run on 1,500 test orbits it had never seen, with activations recorded at eight sites in the network. Each probe was a ridge regression, fitted on 900 orbits, with the penalty and the site chosen on a separate 300, and the R² reported on a final 300 that neither step had touched. A probe could still find structure the model did not use, but it could no longer be rewarded for memorising the orbits it was scored on.

Then came the baselines. For every trained model I probed an untrained twin, a model with exactly the same initial weights that had never seen a training batch. I also probed the raw inputs directly, using the last two positions, and probed for the force computed around four wrong centres, as a lower reference.

The untrained twins reached a Keplerian R² of 0.74. Not 0.98, but still a number that, reported on its own, would have looked like a finding. The raw inputs reached 0.29. The trained models reached at least 0.955 in every condition, and decoded the force with R² between 0.54 and 0.79 against 0.24 for the untrained twins. So the models had learned something real, and the gap between 0.955 and 0.74 is what showed it. The 0.955 alone would not have.

## Why this mattered for everything after

The actual question of the study was never whether these models represent orbits. It was whether changing the training data changes which representation they build: whether sudden kicks to the planet push a long-context model toward the local, force-based picture. The answer turned out to be yes, with force decoding rising from 0.60 to 0.74 at the middle noise level, in every one of five seeds. A difference of that size is only worth reporting if the probe is not inflating everything it touches. Without the controls, I would have had the same comparison sitting on top of a number I could not defend.

So the paper now recommends, plainly, that probe studies on models like these always report an untrained comparison and always evaluate on held-out data. That is not a sophisticated recommendation. It is the kind of thing everyone agrees with in the abstract and skips under deadline, and I understand why, because the uncontrolled number is always the more flattering one.

I wrote in the [previous post](/2026/10/09/held-out-from-what/) that a score needs a comparison that would also score high if the model had understood nothing, and that a random test split quietly fails to provide one. This is the same lesson from a different direction. There the missing comparison was a seasonal split. Here it was an untrained twin. The instinct is the same: before trusting a number, build the version of the experiment in which the model knows nothing, and check that the number falls.

A probe that passes before training begins is grading the probe, not the model.

---

*V. Sai Abhiram is a computer science graduate and software developer based in Hyderabad, working on healthcare technology systems. He writes about machine learning evaluation, the systems that surround models in production, and the research papers behind both. He is the author of two published papers on predictive modelling and spurious correlation in data-driven systems, and of a preprint on the world models transformers learn, with [code and results on GitHub](https://github.com/abhiramvissa/clean-orbits-kepler).*
