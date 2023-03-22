---
title: Using Technical Debt as your next Tool
datePublished: Sun Mar 03 2019 12:00:00 GMT+0000
slug: using-technical-debt-next-tool
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678965803781/CEBfFpFHA.jpg?auto=compress
tags: programming,project-management,software-architecture,qa
domain: caioferreira.dev
---

# Quick Summary

In this post I will show you what is the debt that we collect with during the software lifecycle, what are its causes and how to pay it back.

# Introduction

Often we are faced with a dilemma in software development: implement the best solution for the feature or delivery it quickly but assuming some workarounds and code smells? Whichever side you choose, will be a cost.

This cost is even greater if you work on legacy projects or high changing environments. I am a software engineer @ B2W, the biggest e-commerce on Latin America, and because of its long history, we deal with +6 years legacy applications, in which dozens of developers worked through this time. Sum up the e-commerce environment with its high competitiveness that drives constant changes and new features then you will have a millionaire decision at your hands.

Hence, understating how to manage this cost can be a key asset to achieve a product that has both high quality and a short time to market.

# The decision cost

To understand how to manage the decision cost, we must first learn what cost is. Every decision we make in a software project, not just about best solution vs quick & dirty, incurs in a time cost. Sometimes we cannot take this time and need to target production rather earlier than later. In this situation, we exchange bad code for time. Apply this behavior through time and we get a large debt.

Therefore, **Technical Debt is a metaphor in software development, based on the financial debt, about the accumulation of low-quality code in a project over time**.

Extending the metaphor, we can see financial debt as a resource uptake from some institution, therefore, in the software scope, the resource we borrow is time instead of money and the institution that provides us with this is our code base rather than a bank. Thus, in technical debt calculation, the main amount is the time cost to refactor the codebase to a clean design and the interest represents the extra time that will be taken in the future if the team has to work on the messy code.

# Even more debt

Although the best solution vs quick & dirty decision is an excellent example to introduce technical debt, it isn't its only source. As Fowler (2009) [points out](https://martinfowler.com/bliki/TechnicalDebtQuadrant.html), this case is just a Prudent and Deliberate debt, where the team is aware of the cost it is incurring and know how to pay it back. But we can have four types of debt in total.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678965851191/MMlnGm0h1.png?auto=compress)

Each debt type has a source associated with it:

- Prudent & Deliberate - the team know how to build good software and has the opportunity to deliver more product value if they speed up. They analyze the tradeoff and judge that cost/earn ratio is enough to justify the bad code, otherwise delaying the solution. Then, they plan how to refactor the debt and implement it as early as possible.
- Reckless & Deliberate - the team has the skill to build a well-designed solution but do not has the tools or support to do so. This happens when a team is purposely taking debt but without a plan to repay it or to do so in a code that probably will never change. This scenario is more common on unmotivated teams, legacy projects and due to management pressure for fast delivery instead of a constant and quality delivery.
- Reckless & Inadvertent - the team doesn't know good design practices and incur on costs due to lack of training, experience, and leadership.
- Prudent & Inadvertent - it is a common case that after we delivery a feature or project we realize that the best design approach would be other than what we did. This situation is unpredictable and can happen to any team due to lack of domain knowledge, obscure requirements, and technology limitations.

Of course, this exploration does not exhaust the possible causes for technical debt but includes the most frequent ones.

# Living on high debt

Since the debt causes can be so many, projects can experience an ever-increasing technical debt, to the point where adding new functionality, fix current bugs and operate the application becomes impossible.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1678965872853/gLv1Yjwzc.png?auto=compress)

This type of scenario can be catastrophic, because while the team lives with this application they will lose agility, will observe an increasing bug count, have loss of motivation, increased stress, long production problems (long living bugs), customer complains and possible single points of failure in case only a few people know how to develop and operate the project.

When an application reaches this point we say that it went into technical bankruptcy.

However, using the right strategies we can make the highlighted curve steeper, leading the actual curve nearest to the idealized curve.

# Paying back

In order to avoid that applications achieve high debts or to handle projects already with high debt, we need to build a repayment plan. This can have any sort of effort in order to refactor and pay the time due, but the industry set some strategies that have proven effective.

1. **Technical Backlog**: each task should have a brief description, the reason that the change is important for the project and which part of the code base must change. Like any other task, we should estimate the effort needed to build a good and clean solution. When estimating the cost, the team should add an interest cost that is directly proportional to the probability of this code to change in the future and hence prioritizing the most costly task. A precise estimation is really difficult but a rudimentary approximation is enough to guide the decisions.
   - With this approach the technical debt becomes visible to all stakeholders and the decision of doing such a task can be made upon effort and future impact.
   - The task cost is easily traceable.
   - Don't mix technical and feature tasks.
2. **Refactoring costs included on the feature estimation:** explain the costs is necessary since we follow the principle that no feature should be implemented above bad code and therefore this code should be refactored first (_Boy Scout Rule_). This is a fundamental principle once bad code already incur in interests and adding a new feature using it will increase this interest exponentially, to the point where the cost to work on this code will be so high that will be impossible to make further changes.

**_Besides that, there are two other critic strategies that can be used depending on how high debt the project is in._**

1. **Buffer Tasks:** the task has a portion of the team's sprint. It can be used to unplanned refactorings, unforeseen problems, discoveries, etc. To avoid the task to be wasted on non-relevant work members of the team should always propose how they pretend to use the time and discuss with the rest of the team.
2. **Clean-up Releases:** periodically or in critic scenarios, a team can make releases only with technical refactorings. This strategy is only useful if there is already a refactoring list to be made. Furthermore, the business team should give support as this will probably delay new features. The teams should consider using this when a greater effort is necessary, like in big architectural changes, infrastructural changes, build & deploy, etc.

# Conclusion

Therefore, Technical Debt can be key tool from the product perspective. It allows business people to improve time to market and customer satisfaction with the cost of bad code.

However high debt can bring to huge problems to the project, hence we must strive to be Prudent and Deliberated about our debt, always taking it with a repayment plan together, using all kinds of strategies to keep the debt under control.

To achieve this all stakeholders must be aware of the debt's nature and existence and bring the development team together in the decision.

# References

- [https://www.infoq.com/presentations/debt-aware-culture](https://www.infoq.com/presentations/debt-aware-culture)
- [https://martinfowler.com/bliki/TechnicalDebtQuadrant.html](https://martinfowler.com/bliki/TechnicalDebtQuadrant.html)
- [https://www.infoq.com/minibooks/emag-technical-debt?utm_source=minibooks_about_TechnicalDebt&utm_medium=link&utm_campaign=TechnicalDebt](https://www.infoq.com/minibooks/emag-technical-debt?utm_source=minibooks_about_TechnicalDebt&utm_medium=link&utm_campaign=TechnicalDebt)
- [https://books.google.com.br/books?id=pKVFDwAAQBAJ&lpg=PA18&ots=Gc9vC8vf80&dq=Software Quality Assurance Pressman 2014&hl=pt-BR&pg=PA18#v=onepage&q=Software Quality Assurance Pressman 2014&f=false](https://books.google.com.br/books?id=pKVFDwAAQBAJ&lpg=PA18&ots=Gc9vC8vf80&dq=Software%20Quality%20Assurance%20Pressman%202014&hl=pt-BR&pg=PA18#v=onepage&q=Software%20Quality%20Assurance%20Pressman%202014&f=false)
