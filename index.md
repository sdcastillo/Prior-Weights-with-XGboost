---
layout: default
title: Prior Weights with XGBoost
description: A method of adding prior weights to XGBoost with base_margin.
samwiki: true
---

XGBoost starts from a constant base score and adds trees that follow the gradient of the loss. Under the binary logistic objective that base score is a probability of one half, written on the logit scale. In a sparse region of the features the trees see too few rows to justify a split, so the score for those rows stays near the base value. A prior probability is a deliberate starting point for those rows, chosen before any tree is grown.

The entry point in XGBoost is `base_margin`, a per-row offset already on the link scale. The booster adds that offset to the sum of the tree outputs, then applies the sigmoid, the softmax, or the log-mean. Instance weights, stored on the same `DMatrix` under `weight`, scale how much a row contributes to the gradient and the Hessian. The margin moves the initial prediction. The weight changes the influence of the residual. Setting one does not do the work of the other.

For binary logistic regression, a prior probability π in (0, 1) is stored as the log-odds log(π / (1 − π)). For a Poisson or gamma objective with a log link, a prior mean μ is stored as log(μ), and an exposure offset uses that same log. Training then matches a GLM with an offset: the margin stays fixed, and the trees fit whatever residual variation remains. A large sample can pull the score away from the prior. A thin sample leaves the prior in the prediction, which is the credibility blend already familiar from ratemaking.

Put the margin on the training matrix before `xgb.train`. In the R package that call is `setinfo(dtrain, "base_margin", margin)`. Build the same margin for the rows you score, and attach it to the prediction matrix before `predict`. The per-row margin takes the place of the global `base_score`, so leave `base_score` at its default once the margin carries the prior. Values of π at 0 or 1 make the logit infinite; keep the prior inside the open unit interval.

Places this earns its keep:

- Annual healthcare cost, with last year’s allowed amount as the offset, so a member who barely appears in the current year starts from a known baseline.
- Insurance premium adjustment, with the manual premium as the offset, in the same role an offset plays in a ratemaking GLM.
- Rare attributes. In the agaricus mushroom data shipped with XGBoost, a musty odor covers under half a percent of the training rows, a few dozen examples. A domain probability on that slice gives the booster a starting rate when the leaf would otherwise fall back to one half.
- Stacking, with another model’s predicted probability converted to a log-odds margin, so the new trees are trained to correct that model.
