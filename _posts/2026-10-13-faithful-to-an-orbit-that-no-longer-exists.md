---
layout: post
title: "Faithful to an Orbit That No Longer Exists"
date: 2026-10-13
image: /assets/img/kick-rollouts.png
image_alt: "Three planetary orbits after a kick. The model trained on clean orbits keeps following the old path. The model trained with kicks bends toward the new one."
image_caption: "Three test orbits after a kick. Each model sees the true history, including three positions after the kick, then predicts on its own. The dashed line is where the old orbit would have gone. The model trained on clean orbits follows it; the model trained with kicks bends toward the new path. Panel a of Figure 4 from the paper."
---

Take a planet fifty steps into its orbit and shove it. Change its velocity by a fifth of its speed, in a random direction, and let it carry on along whatever new ellipse that produces. Then show the first few positions of the new path to a model that has been predicting the planet all along, and see what it does next.

In my study, the models trained on clean orbits did something I found hard to look away from. Given three positions on the new path, they went on predicting the old one. Their error was still climbing five steps later, peaked near 0.3, and stayed high for the rest of the orbit. They never recovered, at any noise level, in any of five seeds. They were not confused. They were faithful, to an orbit that no longer existed.

## Two ways to predict a planet

There are two ways to say where a planet goes next. You can watch it for a long time, find the ellipse it traces, and extend the ellipse forward. That is Kepler's way, and it needs a long memory but no idea of forces. Or you can take the current position and velocity, work out the pull of gravity, and step forward. That is closer to Newton, and it needs only the most recent state.

[Liu and colleagues at Stanford](https://arxiv.org/abs/2602.06923) showed earlier this year that small transformers trained on simulated orbits can learn either. With a context of 100 positions, linear probes found the orbital elements of the ellipse in the model's activations. With the context cut to the last two positions, the probes found the force instead. I reproduced the transition with their code before touching anything: at context 2 the force was decoded far better than the ellipse, by context 10 the order had reversed, and at context 100 the Keplerian error was roughly 150 times lower than at context 2. They concluded that locality has to be built into the architecture, by restricting what the model can see.

I wanted to know whether the data could do the pushing instead.

## Why data should matter

There is an old argument from estimation theory about how much of the past is worth remembering. If your measurements are noisy, a long memory helps, because averaging many observations cancels the noise. If the thing you are tracking is sometimes jolted, a long memory hurts, because old observations describe a state that is no longer true. The Kalman filter makes this exact for a simple model: the optimal memory grows with measurement noise and shrinks with the size and frequency of interventions.

The orbits in the original work were clean. Nothing disturbed a planet once it started moving, and the only corruption was noise added to the inputs during training. Under that regime a long memory is the right choice, and a Keplerian model is what a sensible learner should build. So I asked a question the filtering argument makes precise: if the planets were kicked during training, would the models become more local, with the architecture held fixed?

## Seventy-five small transformers

I kept the original architecture, a two-block transformer with a single attention head and 25,634 parameters, and the context of 100 positions. I varied two things in the data: the level of input noise, at three settings, and the rate of velocity kicks, at four, from none up to a kick after roughly six percent of steps, which gives about six kicks per orbit. Every combination was trained with five seeds, and runs with the same seed started from identical weights and saw the same orbits, so each comparison between conditions could be made within a seed. Five more models with a context of two positions served as the reference for what architecture alone can do.

Then every model got the same test: one kick after step 50 on 500 orbits, and a measurement of how long its excess error took to fall back to a quarter of its first value.

Kicks in training changed everything about recovery. At the middle noise level, models trained without kicks never recovered within the orbit. Models trained with the lowest kick rate recovered in about 30 steps, the next in 20, and the highest in 16, and the ordering was identical in all five seeds. At low noise the same three settings recovered in 18, 10 and 7 steps. The rollouts show what the numbers mean: the kick-trained models bent away from the old orbit toward the new one, not perfectly, but unmistakably.

The representations moved the same way. At the middle noise level, the force became easier to read out of the activations, with R² rising from 0.60 for the models trained without kicks to 0.74 for those trained with the most, and it rose in every seed. The ellipse became correspondingly harder to decode. Attention changed too: the share of attention on the five most recent positions rose from 0.38 to 0.61, and where the clean-orbit models kept a flat tail of attention over the whole visible history, the kick-trained models let it fade with distance.

Noise pushed the other way, as the filtering argument said it should. At every kick rate, more input noise produced more Keplerian representations and slower recovery; at the highest noise level, the lowest kick rate was not enough to produce any recovery at all.

The theory gave one more prediction that I could test directly. It says what should matter is the product of the kick rate and the squared kick size, not the number of kicks. So I trained models with half the kick size and four times the rate, and compared them with their matched partners. The pair at the higher rate agreed on every measure I had fixed in advance: recovery in 19.8 steps against 20.2, force decoding of 0.737 against 0.733. Four times as many kicks, and the same behaviour.

## What went wrong

Two of my measures did not do what I predicted, and I report them the way they came out, because the definitions were fixed in a log before any result existed.

Attention was supposed to reach further back as noise increased, since averaging is the point of a long memory. It did the opposite, at every kick rate. The most likely reading is that attention weights are a poor measure of how much of the past a model actually uses, which others have argued before, and which I now believe more than I did. And a measure I built to test whether models remember the pre-kick orbit turned out to mix two things: whether a model notices a kick, which only kick-trained models learn to do, and how long it holds on to the old orbit afterward. Split by time since the kick, the prediction mostly held late and failed early.

There are limits beyond that. The context-2 models, the ones made local by architecture, stayed more local than any model I trained with data: force decoding of 0.92 against 0.79 for my most local long-context model. The data can move a model in a chosen direction; the architecture sets how far it can go. These are 25,000-parameter models on synthetic orbits in two dimensions, and I do not claim that the kick-trained models learned Newton's law of gravitation, only that they stopped relying on a history that could betray them.

## Beyond planets

The models trained on clean orbits were not wrong about physics. They had seen thousands of orbits and every one of them had kept its shape, so they built the representation that the data rewarded, and it served them perfectly until the first time the world changed. Nothing in their training had told them that orbits can change, and so when one did, they could not see it. The models that had been kicked during training carried a different assumption into the test, and that assumption was the only difference between the two groups.

That is a statement about planets, and it is also a statement about every model trained on a history and asked to run in a present that might not resemble it: a model built on one hospital's patients and deployed in another, a forecast trained on years that all looked alike. A model that has never seen the world change will not notice when it does. It will do what my clean-orbit models did, and keep predicting, with complete confidence, a world that is already gone.

---

*V. Sai Abhiram is a computer science graduate and software developer based in Hyderabad, working on healthcare technology systems. He writes about machine learning evaluation, the systems that surround models in production, and the research papers behind both. He is the author of two published papers on predictive modelling and spurious correlation in data-driven systems, and of a preprint on the world models transformers learn, with [code and results on GitHub](https://github.com/abhiramvissa/clean-orbits-kepler).*
