---
title: "GSoC Part 2 - Construct Regression functions in queries with GROUP BY"
date: 2022-06-16T18:18:51+02:00
tags : ['GSoC', 'MariaDB', 'Open Source']
---

# Overview
In this part, I finished the design of regression function, and implemented most part of them. Basically, the implementation of window can be completed at the same, but the tests still need to be verified.

You can see my pr [here](https://github.com/MariaDB/server/pull/2148)
# Design
The core part of these functions is how to accumulate the value instead of calculating at the end of process. I adopted the [Welford's online algorithm](https://www.wikiwand.com/en/Algorithms_for_calculating_variance#/Online) and implement it in `Regr_result` class. Then all of `Item_sum_regr_XXX` will use it to get the value by invoking individual `bool val_real()`. The following UML figure shows the architecture.
{{< mermaid >}}
classDiagram
  Item_sum_double <|-- Item_sum_regr_sxx
  Regr_transition <--o Item_sum_regr_sxx
  Item_sum_regr_sxx <|-- Item_sum_regr_sxy
  Item_sum_regr_sxx <|-- Item_sum_regr_syy
  Item_sum_regr_sxx <|-- Item_sum_regr_avgx
  Item_sum_regr_sxx <|-- Item_sum_regr_avgy
  Item_sum_regr_sxx <|-- Item_sum_regr_slope
  Item_sum_regr_sxx <|-- Item_sum_regr_r2
  Item_sum_regr_sxx <|-- Item_sum_regr_intercept

  Item_sum_regr_sxx: Regr_transition transivalues
  Item_sum_regr_sxx: bool add()
  Item_sum_regr_sxx: void reset_field()
  Item_sum_regr_sxx: void update_field()
  Regr_transition: double m_x
  Regr_transition: double m_y
  Regr_transition: double sx
  Regr_transition: double sy
  Regr_transition: double sxx
  Regr_transition: double syy
  Regr_transition: double sxy
  Regr_transition: ulonglong N

  Regr_transition: void recurrence_next(double nr_y, double nr_x)

  Item_sum_regr_sxy: double val_real()
  Item_sum_regr_syy: double val_real()
  Item_sum_regr_avgx: double val_real()
  Item_sum_regr_avgy: double val_real()
  Item_sum_regr_slope: double val_real()
  Item_sum_regr_r2: double val_real()
  Item_sum_regr_intercept: double val_real()
{{< /mermaid >}}
# Implementation
You could see my code [here](https://github.com/MariaDB/server/pull/2148)

It is worth noting that the aggregation function with `GROUP BY` used the `Item_regr_XXX_field` to store the value. While the aggregation without `GROUP BY` used the `Aggregator` class to iterate each row.

<!-- # What should I do next step?
- How window function works in queries
- How aggregation works in the columnar engine
- Add more tests with edge cases -->


