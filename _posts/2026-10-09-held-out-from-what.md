Layout

post

Title

Held Out From What? Why Random Test Splits Flatter Your Model

Date

2026-10-09

Two numbers from the same study, produced by the same kind of model on the same dataset: an R² of 0.9978 and an R² of 0.5510.

What separates them has nothing to do with the algorithm. Both are Random Forests fitted to Central Pollution Control Board pollutant readings for Delhi. The first was trained on a random shuffle of the days and scored on the days left over. The second was trained only on winter and scored on the rest of the year. Same dataset, same method, different decision about where to draw the line between what the model sees and what it gets judged on.

The study was mine, published earlier this year, and the confounder was there by design. I built the target, a Respiratory Risk Index, with seasonal weighting deliberately mixed in, so I would know in advance what the model was not supposed to lean on. What I did not anticipate was how completely the standard evaluation would cover for it. Under the shuffle, the model looked close to perfect. It had not learned much about how air quality relates to respiratory risk. It had learned to recognise winter.

That matters well beyond one air quality dataset, because the shuffle is the default. train_test_split in scikit-learn shuffles unless you explicitly stop it, and almost every tutorial you have worked through opens that way. The line of code reads like a formality. It is actually a claim, made on your behalf, about the world the model is going to be dropped into: that the future will be a reshuffled version of the past.

What the shuffle actually guarantees

A random split draws test rows from precisely the same joint distribution as the training rows. Whatever confounders live in the training data live in the test data too, in roughly the same proportions and standing in the same relationship to the target. That is not a flaw in the procedure. It is the procedure working as designed.

Follow that through with the seasonal case. If winter drives both pollution levels and the target, a randomly held-out day in January has hundreds of close neighbours in the training set, all from other Januaries, all carrying the same seasonal fingerprint. The model never has to learn anything about particulates and lungs. It only has to recognise the season. And the test set has no way to catch it, because the test set is made of winter too.

So held-out data is not really held out. It is held out from the fitting procedure, which protects you against memorising individual rows, and that is a genuine risk worth protecting against. It is not held out from anything else.

This is not a niche worry

The cleanest demonstration in the wild comes from John Zech and colleagues, writing in PLOS Medicine in 2018. They trained pneumonia-screening networks on 158,323 chest radiographs drawn from three hospital systems and found external performance lower than internal in three of five comparisons. The detail that explains why sits further down the paper: the networks could identify which hospital system an image came from with better than 99.9 percent accuracy. A model that recognises the hospital can infer that hospital's disease rate without ever attending to a lung.

The pandemic produced a blunter version of the same story. Alex DeGrave, Joseph Janizek and Su-In Lee pulled apart high-performing COVID classifiers built on chest radiographs and showed they were reading acquisition artifacts and dataset provenance rather than pathology, which is why they looked accurate at home and collapsed at new hospitals. That was not an isolated case. A systematic review led by Michael Roberts screened 2,212 studies, kept 62 after quality filtering, and concluded that not one of the models was of potential clinical use, with weak validation among the recurring flaws. Thousands of papers, and effectively all of them reporting good numbers.

You do not even need a confounder for the split to flatter you. In 2019, Benjamin Recht, Rebecca Roelofs, Ludwig Schmidt and Vaishaal Shankar rebuilt fresh test sets for CIFAR-10 and ImageNet, following the original collection protocols as faithfully as they could, then re-ran a wide range of published models on them. Accuracy fell by 3 to 15 points on CIFAR-10 and 11 to 14 on ImageNet, which on ImageNet they put at roughly five years of research progress. Their own explanation is the part worth sitting with: the drop was not caused by years of quietly overfitting to a reused test set, but by the new images being slightly harder. A careful, deliberate attempt to resample the same distribution still introduced a shift nobody could see coming.

When the shift is structural rather than accidental, the gap becomes systematic. The WILDS benchmark, assembled by Pang Wei Koh and a large group of collaborators, gathers ten datasets built around shifts that occur naturally in deployment: different hospitals, camera traps, time periods, countries. Standard training scored substantially worse out-of-distribution than in-distribution on every single one, and the robustness methods available at the time did not close the gap.

Splitting along the thing you are afraid of

None of this argues for abandoning held-out evaluation. It argues for choosing the split deliberately instead of inheriting it, and a few habits follow from that.

The first is to name the shift before you split anything. One sentence is enough. This model will score patients at clinics we have never seen. This model will run on next quarter's transactions. This model will be used in summer. Then cut the data along that axis and hold out a site, a period, a device, a season, a cohort. If you cannot name the shift, you have learned something useful about the project before writing a line of code.

Naming it also guards against a quieter failure: leakage at the level of the entity rather than the row. Random splitting invites the same patient, user or match to land on both sides of the line, and once that happens the score means nothing, however well behaved the distribution is. Scikit-learn ships GroupKFold, StratifiedGroupKFold and TimeSeriesSplit for exactly this, and each costs one line. The Roberts review makes a point of saying authors should have to state explicitly how they kept one patient's images out of both partitions, which tells you how rarely anyone does.

Once you have both splits, report both scores. A single number invites the reader to assume the best case; two numbers force a conversation about which case they are actually in. On its own, 0.9978 is a claim about a model. Sitting next to 0.5510, it becomes a claim about the world, and a far more honest one. The gap is the finding, not the embarrassment.

What the gap will not do is explain itself, and this is where feature importance tends to get misread. In my second configuration, PM2.5 carried more weight than AQI itself, which told me where the model was leaning but not why it would fall over. Only the seasonal test told me that. Importance rankings describe the model you happened to fit, not the process that generated the data, so they will put a confounder at the top of the list without any warning that they have done so.

The harder case is when you cannot observe the confounder at all, and that is where building one earns its keep. The Respiratory Risk Index in my study was a proxy, a real limitation of the work. But constructing the confounding on purpose meant I knew what the model was not supposed to learn, and could check whether the evaluation noticed. That is the cheap version of this whole idea: plant a shortcut, then see whether your test catches it. An evaluation that misses a shortcut you buried yourself will certainly miss one you did not.

Two objections are worth taking seriously before any of this hardens into a rule. The first is that shifted evaluation makes results look worse, sometimes unfairly. If a model really will be retrained every week on fresh data from one site, the deployment is close to independent and identically distributed, and the random split is asking the right question. Evaluation should mirror deployment, and deployment is not always shifted.

The second is that a large gap is not by itself a verdict on the model. It can mean the shift is simply hard. It can mean the target changed meaning between environments. It can mean winter and summer are two different problems and asking one model to cover both was the mistake, which is at least partly what happened in my case. The gap opens a question rather than settling one. But you have to be able to see it before you can ask it.

Which is the argument, in the end. Any score you publish is a prediction about a situation someone else will be standing in. A random split predicts that tomorrow is a shuffled version of yesterday, and often enough that holds. When it does not, the shuffle will not tell you. It will hand you a number you cannot defend once somebody acts on it.

V. Sai Abhiram is a computer science graduate and software developer based in Hyderabad, working on healthcare technology systems. He writes about machine learning evaluation, the systems that surround models in production, and the research papers behind both. He is the author of two published papers on predictive modelling and spurious correlation in data-driven systems, and of a preprint on the world models transformers learn, with code and results on GitHub.
