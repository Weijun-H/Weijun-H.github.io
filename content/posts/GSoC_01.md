+++
title = "GSoC Part 1 - Explore the Mechanism in MariaDB"
date = "2022-06-11T19:53:44+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used.

tags = ['GSoC', 'MariaDB', 'Open Source']
+++
## Overview
My task is to implement regression functions in MariaDB, which was mentioned in [MDEV-17467](https://jira.mariadb.org/browse/MDEV-17467). This feature has been implemented by many MariaDB competitors, like PostgreSQL, Oracle, and so on.

You can see my pr [here](https://github.com/MariaDB/server/pull/2148)

## Tasks
The functions I need to implement are following:
- `REGR_SLOPE`: Returns the slope, equal to `REGR_SXY` / `REGR_SYY`
- `REGR_INTERCEPT`: Returns the y-intercept of the regression line, which is
equal to `REGR_AVGX` - `REGR_SLOPE` * `REGR_AVGY`
- `REGR_COUNT`: Returns the number of non-empty pairs of numbers used to
fill the regression line
- `REGR_R2`: The return value is equal to (`REGR_SXY` * `REGR_SXY`) /
(`REGR_SYY` * `REGR_SXX`)
- `REGR_AVGX`: calculate the mean of the independent variable (expr2) of the
regression line, after removing the null pair (expr1, expr2), is equal to
`AVG(expr2)`
- `REGR_AVGY`: Calculate the mean of the strain variable (expr1) of the
regression line, after removing the null pair (expr1, expr2), which is equal to
`AVG(expr1)`
- `REGR_SXX`: The return value is equal to `REGR_COUNT` * `VAR_POP(expr2)`
- `REGR_SYY`: The return value is equal to `REGR_COUNT` * `VAR_POP(expr1)`
- `REGR_SXY`: The return value is equal to `REGR_COUNT` * `COVAR_POP(expr1, expr2)`

## Mechanism
### How to define a function in MariaDB
For **native functions**, it is easy to define. All native functions are defined in `sql/item_create.cc`. You can add function definitions there, parser can successfully recognize them.

In terms of **non-native functions**, this process is different. You can follow these steps, if you add a new aggregate functions:
- `sql\lex.h`: list of function symbols
- `sql\sql_yacc.yy`: contains the definition of the SQL language subset understood by MySQL
- `sql\item_sum.cc`: list all of aggregate functions

Basically, you could only modify these three files to create a new non-native function.
### How does the aggregate function work?
The typical aggregate function has the following member:
- `add()`: iterate each row and update the result
- `val_real()`: return the final result
- `reset_field()`: before iterating each row, reset the class members
- `update_field()`: update and store
The processes of querys without and with `GROUP BY` are different.

If you use the query without `GROUP BY`, like `SELECT SUM(value) FROM T1`, the parser will find `SUM` and invoke `Item_sum_sum(thd, $3)` constructor. Then the parser goes through each row, and update the result by functions above. But if using `GROUP BY` in the query, the class will call constructor `  Item_sum_sum(THD *thd, Item_sum_sum *item):`. The result will be provided by `Item_sum_field` class.

### How is the value stored?
I used the class `Regression_result` to store all the results related to regression functions. The main components are below

``` cpp
class Regression_result
{
  // store results
  Stddev sxx, syy;
  double sxy;
  double sx, sy;
  ulonglong N;
public:
  // iterate over the results
  void recurrence_next(double nr_y, double nr_x);
  // for window function
  void remove(double nr_y, double nr_x);
};
```

### How could we iterate the value instead of calculating at the end of the process?
The difficult point is to calculate `sxy`, `syy` and `sxx`. Because they are equivalent to variance and covariance, I adopted [Welford's online algorithm](https://www.wikiwand.com/en/Algorithms_for_calculating_variance#/Online) to calculate them. I defined `recurrence_next` to implement it.
```cpp
void Regression_result::recurrence_next(double nr_y, double nr_x)
{
  if (N++)
  {
    sx+= nr_x;
    sy+= nr_y;
    double diff = nr_x - sxx.get_m();
    sxx.recurrence_next(nr_x);
    syy.recurrence_next(nr_y);
    sxy= sxy + diff * (nr_y - syy.get_m());
  }
  else
  {
    sx= nr_x;
    sy= nr_y;
    sxx.recurrence_next(nr_x);
    syy.recurrence_next(nr_y);
  }
}
```
In this way, the result can be calculated iteratively.

On the other hand, if we only want to efficiently calculate the result in window function , like
```SQL
REGR_AVGX(age, weight) OVER(ORDER BY id rows between 2 preceding and 2 following) as avgx
```
the result need to remove the first one in order to calculate the next. The implementation is similar to the recurrence_next
``` cpp
void Regression_result::remove(double nr_y, double nr_x)
{
  if (N > 0)
  {
    sx-= nr_x;
    sy-= nr_y;
    sxx.remove(nr_x);
    double diff = nr_x - sxx.get_m();
    sxy= sxy - diff * (nr_y - syy.get_m());
    syy.remove(nr_y);
    N--;
  }
}
```

## What should I do next step?
- Figure out memory management in aggregate functions
- Complete all functions with and without `GROUP BY`.