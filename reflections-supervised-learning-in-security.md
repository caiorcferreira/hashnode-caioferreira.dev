---
title: Reflections about Supervised Learning on Security
slug: reflections-supervised-learning-in-security
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682263607494/idfExTZ29.jpg?auto=format
tags: ml,security,supervised-learning,ai
domain: caioferreira.dev
---

Supervised learning is a technique that aims to learn a hypothesis function _h_ that fits a behavior observed in the real-world, which is governed by an unknown function \\(f\\).

To learn this function, we use a set of example data points composed of inputs (also called features) and outcomes (sometimes called labels). These example data points were sampled from the real world behavior, i.e. from the function \\(f\\), at some time in the past. Our goal is that we can extract knowledge from the past to figure out this behavior on unseen data beforehand. This knowledge extracted from the set of examples is materialized on the \\(h\\) function.

In Security, we can imagine some examples where this would be useful, like trying to learn if an HTTP request contains malicious payload or if some set of bytes is a malware or not. However, supervised learning is less often used in Security than in many other domains.

This happens because the principles of the supervised learning theory conflicts with the nature of Security, limiting its application. But, by understanding these principles, it is also possible to see how to best apply this technique and how it can maybe useful.

## Stationary assumption

The most important principle in supervised learning is the stationary assumption. When the data that represents the real world behaviors follows the stationary assumption, it means that predicting the behavior using past example is approximately correctly.

The **stationary assumption** states that the behavior that is being learned don't change through time. This has some important consequences:

1. We expect that each data point is independent of each other. This is important, because if there were causal effects between data points, then the features of a data point \\( x_{n+1} \\), caused by \\( x_{n} \\), would vary with a probability that is a combination of the probability distribution and the effect of \\( x_{n} \\), hence the distribution would not remain the same over time, because \\( x_{n} \\) and \\( x_{n+1} \\) would vary in different ways. For example, in a box with 1 blue ball and 2 red balls, the probability of picking a blue ball when drawing the first one from the box is 33% while a red one would be 66%. However, if the first one is indeed blue, then the probability of drawing a red ball as the second one is 100%. The first data point (blue ball draw) changed the probability of the second data point.
2. We expect that each data point is identically distributed, i.e. each data point should be drawn from the same probability distribution. We could learn the shopping behavior using data ranging from Black Friday to New Years, however the users' behavior in this time is completed different from the rest of the year, therefore the data used to learn has a different probability distribution than the unseen data on which we are going to make predictions.

Any dataset that follows these two characteristics is said to hold the i.i.d assumption (independent and identically distributed). The importance for our training datasets on supervised learning to be i.i.d is because it connects the past to the future, without it, any inference made on the available data would be invalid.

Understand the i.i.d assumption is specially important for Security Machine Learning because it is one of the areas where causality and behavior shifts are most present. So, exactly how this assumption affects our ability to do Supervised Machine Learning?

### Causality

Many threat behaviors have causal nature, and therefore we should have a lot of care when preparing our datasets and choosing our validation methods.

A good example is malware classification, where you could have many samples from various families from different years. Each family generation influences each other, and sometimes they have similar characteristics.
A special bad situation that could happen with this is that during splitting of the dataset between training and testing, without taking into account the time relation of the families and samples, then you could end up training the model with future information and testing against past samples. This would produce falsely accurate results that would not generalize in the real-world.

There are ways to deal with this, but it depends on the type and strength of the causal relation between the data. For this case, ensuring that the newest malware samples are used for test should be enough.

### Behavior Shift

We could create a model to learn a threat behavior like the profile of a botnet, however once we started responding effectively to it, adversaries would adapt and our model would become useless because the new botnets would have a totally different behavior.
This would be the case of a change in the probability distribution from which the features are drawn, leading to the break of our assumption.

## Looking to the other side

Although causality may be addressed by good data preparation, preventing a model to be become outdated due to behavioral shift is almost impossible. However, this problem isn't a new one in Security, such that one best practices is to instead of trying to detect and block malicious action, defenders should define what a legitimate system behavior looks like and block everything else. This can be summarized as: allow lists are more secure than block lists.

We can apply the same philosophy for supervised learning, by modeling profiles of legitimate behavior, which usually are more stable. Then, new data points are classified against these multiple profiles models, finally a meta-classifier is used to choose which one is the best fit or if it's an outlier. This has the ability to catch any new threat behavior that deviates from the know legitimate profiles.

This combination of multiple models is called ensemble learning, which has shown to improve models performance, like when comparing a Decision Tree model to a Random Forest one.

As a side bonus, since this way of building models does not depend on knowing threat behaviors, it avoids the common problem in mining data for Security that usually produce highly unbalanced datasets. We can train each profile using the true positive data of others profiles as false examples, assuming each profile is mutually exclusive.

The main challenge in this approach is that adversaries may try to mimic legitimate profiles, however like the authors of Notos showed, this can be useful sometimes, as in their cases this would imply in adversaries using a more stable network infrastructure that would be easily defeated by static block lists. Therefore, forcing adversaries to mimic a legitimate profile could also reduce their capabilities.

## Conclusion

In conclusion, supervised learning is a powerful technique that allows us to extract knowledge from past data points to predict future behavior. However, in the Security domain, applying it comes with unique challenges due to the presence of causality and behavior shifts. Understanding the stationary assumption and the importance of having an i.i.d dataset is crucial, because it informs us how to best prepare our datasets, choose our validation methods and, most importantly, what are the best behaviors to be modeled using supervised learning.

Luckily, by modeling profiles of legitimate behavior, and catching new threat behavior that deviates from known legitimate profiles, we can build intelligent allow lists.

While there are challenges, with proper preparation and understanding, supervised learning can be a valuable tool in Security.

## References

- [Notos: Building a Dynamic Reputation System for DNS](https://astrolavos.gatech.edu/articles/Antonakakis.pdf)
