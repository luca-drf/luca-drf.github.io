---
title: "Taming Floating-Point Arithmetic in Python (Part 1)"
date: 2024-05-26T13:09:37+01:00
tags: [python]
draft: false
featured: true
image: "https://i.ibb.co/SndJ6g0/rob-thompson-om-Uf5-OUJx-Xc-unsplash.jpg"
alt: "'Mind the gap' sign on London's tube platform."
summary: "First of a two parts series illustrating common gotchas in IEEE-754 floating-point arithmetic, with practical and realistic examples using Python's built-in float (double-precision)"
description: "Blog post on common gotchas in IEEE-754 floating-point arithmetic, with practical and realistic examples using Python's built-in float (double-precision)."
---


Introduction
------------

If you're developing software that perform some sorts of numeric calculations,
there's a good chance that you'll have to deal with floating-point arithmetic.

In this blog post I'll show few practical cases where seemingly innocuous
floating-point operations can lead to considerable errors, and how such cases
can be treated in Python with the aid of [decimal](https://docs.python.org/3/library/decimal.html)
and [fractions](https://docs.python.org/3/library/fractions.html) modules.


IEEE-754 Double-Precision Floating-Point format
-----------------------------------------------

The default built-in type used for handling decimal numerals in Python is
[`float`](https://docs.python.org/3/library/functions.html#float) which implements
[IEEE-754 Double-Precision Floating-Point format](https://en.wikipedia.org/wiki/Double-precision_floating-point_format), and its intrinsic pros and cons.

Doubles are represented as the result of a multiplication between a fraction
(significand) and an exponent and stored in a fixed length binary word of 64
bits (1 bit of sign, 11 bits of signed integer exponent and 52 bits of significand).

![Floating-Point Double-precision format](https://i.ibb.co/DYvLDK5/Screenshot-2024-05-26-at-13-32-39.png?w=800)

![Floating-Point Double-precision value](https://i.ibb.co/yWgYs80/Screenshot-2024-05-26-at-13-33-42.png?w=400)

Due to _double's_ format fixed length, real numbers are (in most cases)
represented by approximations (i.e. the nearest representable number by a _double_)
For example 0.1, 0.2, 0.3 and 0.4 can't be represented exactly.
Also, due to the value of a _double_ being the result of a multiplication, _double's_
values aren't evenly spread across the range or representable numbers.

![Floating-Point Double-precision spread](https://i.ibb.co/890XSns/floatingpoints.png?w=800)

From [Wikipedia](https://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64):

> Between 2{{<sup "52">}} (4,503,599,627,370,496) and 2{{<sup "53">}}
> (9,007,199,254,740,992) the representable numbers are exactly __the integers__.
>
> For the next range, from 2{{<sup "53">}} to 2{{<sup "54">}}, everything is
> multiplied by 2, so the representable numbers are __the even ones__, etc.
>
> Conversely, for the previous range from 2{{<sup "51">}} to 2{{<sup "52">}},
> the spacing is 0.5, etc.

Hence, even though the larger absolute value representable with a _double_ is
10{{<sup "308">}}, it makes little sense to use _doubles_ to represent
values greater than 2{{<sup "52">}}.

Although the magnitude of the above numbers might seem "extreme" for "regular"
applications, floating-point approximation can create significant problems
when performing arithmetic operations with numbers at a much lower scale.

**Note**: I haven't mentioned _subnormal_ (or _denormal_) representation as it's
not relevant for the scope of this blog post.


Floating-Points notable gotchas
-------------------------------

### Addition series are not associative

Summing numbers with different orders of magnitude leads to larger approximation
errors than numbers of the same order of magnitude, hence changing the order in
which multiple sum operations are performed also changes the overall result.

Let's analise the following examples:

```python
>>> .9999 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001
0.9999999999999996
```
Note that `1.0` is (obviously) exactly representable with a _double_, however, the
above series of sums returns a _double_ that is close but not exactly `1.0`.

This is because the Python interpreter executes operators with the same
precedence from left to right and the first sum operation that is encountered is
`.9999 + .00001` which is summing two numbers four order of magnitude apart. The
result of this operation is then summed by another `.00001`, again a sum between
two numbers four orders of magnitude apart, and again for the following sums.
So, we have a series of ten sums of different magnitudes and that generates
enough error that the overall resulting _double_ is close but not quite `1.0`.

Summing the number with the same magnitude first returns a more accurate
result. In fact:

```python
>>> .9999 + (.00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001 + .00001)
1.0
```

Note that this time we're summing numbers four order of magnitude apart only
once.

That solves the problem, in this case, but in practice you're much more likely
to encounter series of sums as a result of summing in a `for` loop or when using
built-in `sum()` (or similar third-party implementation) and summing arbitrary
iterables (with arbitrary numbers in them and that you don't know much about).

So you'd have to implement an efficient algorithm to order the numbers by
magnitude before summing, which can be a bit of a pain and likely result in a
non-trivial performance hit.


### Subtracting nearly equal numbers is not precise

Subtractions are also problematic, but unlike additions, problems arise when
subtracting numbers very close to each other. For example, imagine you're
calculating the distance between two pairs of points on a line:

```python
>>> 17.9995 - 17.9975
0.0020000000000024443

>>> 9.9995 - 9.9975
0.0019999999999988916
```

The distance between `17.9995` and `17.9975` and between `9.9995` and `9.9975`
should be the same (`0.002`). However, the results of the two subtractions are
close but far enough that if we were to truncate at the third decimal place we'd
have a result that's half of what it should have been. Or for instance, if we
wanted to trigger some kind of behaviour if the distance between two points was
`> 0.002` then the first pair of points would trigger it, but not the second
despite theoretically have the same exact distance.

In numerical analysis, this phenomenon is called
[_Catastrophic cancellation_](https://en.wikipedia.org/wiki/Catastrophic_cancellation).

At this point, I can almost hear some of you now laughing at the screen
thinking:

_"These aren't real issues. Nobody would be fool enough to compare
floats that way, compare the rounded numbers instead!"_

And you would be absolutely right, in this case. But often the code logic is
much more complex, and you might find yourself not having a fixed threshold but
rather a variable proportion, so you might have to round that number as well
and/or you might not know up to how many decimal places is "safe" to round.


### Rounding

Rounding can be helpful when comparing or storing decimal numbers, it's **always**
preferable than truncating (unless you want to end up as the
[Vancouver Stock Exchange](https://en.wikipedia.org/wiki/Vancouver_Stock_Exchange))
but you should be aware that rounding is an operation that in most cases removes
information, hence precision.

If we consider the previous section's subtractions, Python's `round()` can help
in quantify the results to the significant digits we need. In fact:

```python
>>> round(17.9995 - 17.9975, 3)
0.002

>>> round(9.9995 - 9.9975, 3)
0.002
```
So if rounding the result of the difference to three decimal places returns "the
expected result", as in my reference system I'm always and only interested in
three decimal places, surely the invariant would hold if the number were
previously rounded to the same number of decimal places. Well, most likely, not:

```python
>>> round(round(9.9995, 3) - round(9.9975, 3), 3)
0.001  # Hmmmmm....

>>> round(round(17.9995, 3) - round(17.9975, 3), 3)
0.003  # Wait, what?!
```

Surely `round(9.9995, 3) == 10.0` and `round(9.9975, 3) == 9.998` which differ
by `0.002`. And surely `round(17.9995, 3) == 18.0` and `round(17.9975, 3) ==
17.998` which also differs by `0.002`, right?!

Actually, no. In fact:

```python
>>> round(9.9995, 3)  # 9.9995 is represented by 9.9994999999999993889332472463138401508331298828125
9.999  # Resulting in 9.999 as the fourth decimal place is 4

>>> round(9.9975, 3)  # 9.9975 is represented by 9.9975000000000004973799150320701301097869873046875
9.998  # Resulting in 9.998 as the fourth decimal place is 5

>>> 9.999 - 9.998
0.0010000000000012221  # Which rounded is 0.001
```
Similarly:

```python
>>> round(17.9995, 3)  # 17.9995 is represented by 17.999500000000001165290086646564304828643798828125
18.0  # Resulting in 18.0 as the fourth decimal place is 5

>>> round(17.9975, 3)  # 17.9975 is represented by 17.997499999999998721023075631819665431976318359375
17.997  # Resulting in 17.997 as the fourth decimal place is 4

>>> 18.0 - 17.997
0.0030000000000001137  # Which rounded is 0.003
```
So in the first case the result is half what we expected in the second case is
one third more. Not great, considering that we're not dealing with numbers of
great magnitude in absolute terms. These are very common magnitudes.

On top of the above, is worth remembering that Python's built-in `round()`
implements IEEE-754 default rounding rule, that is "Half to Even". Which means
that when (and only when) the fractional part of the decimal number is equal to
`0.5` then the integer part will be rounded to the nearest even. In fact:

```python
>>> round(4.5)
4

>>> round(3.5)
4

>> round(4.6)
5

>>> round(0.065, 2)  # 0.065 is represented by 0.065000000000000002220446049250313080847263336181640625
0.07
```
And that's rounding for you.

[![xkcd.com/2585](https://imgs.xkcd.com/comics/rounding.png)](https://xkcd.com/2585/)


More complex examples
---------------------

All the above examples are interesting but real-life code is much more complex
than that. As mentioned, it's much more likely to have to deal with operations
executed in series as part of loops or functions, so let's have a look at some
more complex (and subtle) examples.


### The significance eater

Consider the following function:

```python
def significance_eater():
    x = 10.0 / 9.0
    for _ in range(25):
        print(x)
        x = (x - 1.0) * 10.0
```
The idea is to show how, due to `float`'s finite representation, significance
can be lost "gradually" by performing a series of operations on the same number.

Note that we're instantiating `x` to the value of 10/9 or
1.111...{{<sub "periodic">}}, then at each iteration of the loop, we first
subtract 1 from the number, arithmetically resulting in 0.111..., then multiply
by 10, arithmetically resulting in 1.111... or the initial number.

So if `float` representation had unlimited memory, at the end of each iteration
of the above loop, the value of `x` would be unchanged. But `float` is
limited to 64bits so what happens when we run the above code is this:

```python
>>> significance_eater()
1.1111111111111112
1.1111111111111116
1.111111111111116
1.1111111111111605
1.1111111111116045
1.1111111111160454
1.1111111111604544
1.1111111116045436
1.1111111160454357
1.1111111604543567
1.1111116045435665
1.111116045435665
1.11116045435665
1.1116045435665
1.1160454356650007
1.160454356650007
1.6045435665000696
6.045435665000696
50.45435665000696
494.5435665000696
4935.435665000696
49344.35665000696
493433.5665000696
4934325.665000696
49343246.65000696
```

At each iteration some significance is lost and after the 16{{<sup "th">}}
iteration, **all** the initial information is lost.

This behaviour is similar to _Overflow_ and _Underflow_, but it's harder to
spot and troubleshoot.


### Simple variance calculator

Consider now the following a naïve implementation of a function for calculating
the variance of a series of numbers in a single pass:

```python
import numpy as np

def simple_variance(nums):
    sum_of_squares = 0
    sum_of_nums = 0
    N = len(nums)
    for num in nums:
        sum_of_squares += num**2
        sum_of_nums += num
    variance = (sum_of_squares - sum_of_nums**2 / N) / N
    print(f"Real variance: {np.var(nums)}")
    print(f"Simp variance: {variance}")
```
The explanation of the algorithm can be found [here](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance).

Note that the above code prints the variance calculated using the naïve
approach just after the "real" variance calculated with a more sophisticated
algorithm defined in Numpy, so we can visually compare the two.

So if we call `simple_variance()` passing a list of integers, the algorithm
doesn't behave too badly:

```python
>>> simple_variance([2, 7, 3, 12, 9])
Real variance: 13.84
Simp variance: 13.840000000000003  # not bad
```
But when we pass a very large list of large numbers with little variance, then
things gets weird:

```python
>>> simple_variance(np.random.uniform(1_000_000, 1_000_000.06, 100_000))
Real variance: 0.0003006493219911617
Simp variance: 0.02288  # oh no...

>>> simple_variance(np.random.uniform(1_000_000, 1_000_000.06, 100_000))
Real variance: 0.00029977863201331316
Simp variance: -0.00544  # 乁(⊙_◎)ㄏ
```
In the first call above, the error is two orders of magnitude while in the
call we get a negative variance (variance can't be negative)! This is very bad!

The problem here is again _catastrophic cancellation_, and in the above case
that happens when subtracting `sum_of_squares - sum_of_nums**2` as the sum of
squares is very close to the sum of nums squared.

The algorithm mentioned above is very well known for its cancellation problem,
but I think it's a good example to show how hard it can be to spot these kinds
of problems; imagine you had a similar algorithm working relatively fine with
certain sets of numbers and then returning "garbage" when the numbers change!


Conclusions
-----------

All the above is not only strictly related to Python, any language implementing
IEEE-754 Floating-Point arithmetic will have the exact same problems.

In the next chapter we're going to see how to tackle all the above problems
using only Python's standard library modules.


References
----------

If you want to dive deeper into the theory behind the problem exposed in this
post, you should definitely start from these links:

- [Floating Point Arithmetic: Issues and Limitations](https://docs.python.org/3/tutorial/floatingpoint.html)
- [The Perils of Floating Point](http://www.indowsway.com/floatingpoint.htm)
- [Floating Point Computation, Computer Laboratory, University of Cambridge](https://www.cl.cam.ac.uk/teaching/1011/FPComp/fpcomp10slides.pdf)
- [What Every Computer Scientist Should Know About Floating-Point Arithmetic](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html#693)

This blog post was inspired by the following practical guides:
- [FP Issues and Limitations](https://docs.python.org/3/tutorial/floatingpoint.html)
- [Examples of Floating Point problems](https://jvns.ca/blog/2023/01/13/examples-of-floating-point-problems/)
- [Losing My Precision: Tips For Handling Tricky Floating Point Arithmetic](https://www.soa.org/news-and-publications/newsletters/compact/2014/may/com-2014-iss51/losing-my-precision-tips-for-handling-tricky-floating-point-arithmetic/)


Well done for getting here, see you soon in the next chapter!
