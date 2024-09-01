---
title: "Taming Floating-Point Arithmetic in Python (Part 2)"
date: 2024-06-23T16:45:00+01:00
tags: [python]
draft: false
featured: true
image: "https://i.ibb.co/vYkzGmX/joshua-hoehne-AZr-BFo-XP-3-I-unsplash.jpg"
alt: "A young student is solving arithmetical exercises on a piece of paper"
summary: "Second and last part of the series tackling IEEE-754 floating-point arithmetic with Python. In this post we're looking at how Python's standard library Decimal and Fractions modules can help dealing with floating-point gotchas seen in the previous part."
description: "Blog post on how Python's Decimal and Fractions modules can help debugging and/or overcoming limitations of IEEE-754 floating-point specification."
---


Introduction
------------

In the previous [part]({{< ref "/blog/taming-floating-points.md" >}} "Taming Floating-Point Arithmetic in Python (Part 1)")
we explored some practical gotchas that are easily encountered when dealing with
floating-point arithmetic. Luckily,
[IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) is not the only standard for
representing decimal numbers on binary based systems; in the late 80s two
standards defining floating-point arithmetic with radices other than 2
(including radix 10) where published
[IEEE-754-1985 and IEEE-854-1987](https://en.wikipedia.org/wiki/IEEE_854-1987)
and later merged and revised in [IEEE-754-2008](https://en.wikipedia.org/wiki/IEEE_754-2008_revision).

IBM Mainframe [System z9](https://en.wikipedia.org/wiki/IBM_System_z9) was the
first CPU to implement IEEE-754-2008 in its microcode however most CPUs haven't
adopted this standard. On the other hand, this revision is available through
software as it has been implemented for most programming languages.

In Python, base 10 floating-point arithmetic is implemented in the standard
library by the [decimal](https://docs.python.org/3/library/decimal.html) module,
and is based on IBM's [General Decimal Arithmetic Specification](https://speleotrove.com/decimal/decarith.html).


Python's Decimal
----------------

The [decimal](https://docs.python.org/3/library/decimal.html) module provides
support for fast correctly rounded decimal floating-point arithmetic via the
`Decimal` class which presents several advantages over built-in `float`:

 - Represents decimal numbers exactly, and the exactness carries over into arithmetic
 - Incorporates a notion of significant places as arithmetic operations are defined in base 10
 - Unlimited precision
 - Exposes all required parts of the standard to the user (precision, rounding strategy, signal handling can all be modified)
 - Decimal objects are immutable (and _hashable_)
 - Decimal integrates well with other Python’s built-ins (e.g. `round()`)

The Decimal specification represents fixed and floating-point numbers as
triplets consisting of _sign_ (a single bit), _digits_ (an array of unsigned
integers) and _exponent_ (a signed integer). For example the number `123.45` is
represented as:
```
(sign=0, digits=(1, 2, 3, 4, 5), exponent=-2)
```

Python's decimal objects can (and should) be instantiated from strings as using
floats would immediately introduce rounding error and noise. For example:

```python
>>> from decimal import Decimal

>>> Decimal("0.3")
Decimal('0.3')

>>> Decimal(0.3)
Decimal('0.299999999999999988897769753748434595763683319091796875')
```

Note that when instantiating `Decimal` with `float` all the digits of the
floating-point representation are stored losing precision and obscuring the
significance. That defeat the purpose when performing operations:

```python
>>> Decimal("0.3") + Decimal("0.3")
Decimal('0.6')

>>> Decimal(0.3) + Decimal(0.3)
Decimal('0.5999999999999999777955395075')
```


Taming floating-point arithmetic with decimal
---------------------------------------------

In this section we'll see how we can use decimal effectively for solving the
problems seen in part 1.


### Solving imprecise addition and subtraction

Let's consider non-associative addition as seen in this
[example]({{< ref "/blog/taming-floating-points.md/#addition-series-are-not-associative" >}} "Addition series are not associative"); that's not a problem when using decimal's
arithmetic as numbers gets aligned to the highest absolute exponent then
the sum is carried over in columns (like you would do on a piece of paper).
In fact:

```python
>>> Decimal(".9999") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001") + Decimal(".00001")
Decimal('1.00000')
```
The result is precise and the resulting `Decimal` object, stores all the
significant figures (the result is `1.00000` not `1`). Hurrah!

To have look at the decimal tuple stored under the hood, you can use
`Decimal.as_tuple()`:

```python
>>> Decimal("1.00000").as_tuple()
DecimalTuple(sign=0, digits=(1, 0), exponent=-5)
```

The same logic is applied to subtraction, therefore solving the floating-point
imprecision when subtracting nearly equal numbers seen in
[example]({{< ref "/blog/taming-floating-points.md/#subtracting-nearly-equal-numbers-is-not-precise" >}} "Subtracting nearly equal numbers is not precise").


### Helping with rounding

As mentioned before, `Decimal` object works well with other built-ins. Here's
how rounding invariant works as expected instead of breaking when rounding
floats in this [example]({{< ref "/blog/taming-floating-points.md/#rounding" >}} "Rounding"):

```python
>>> round(round(Decimal("9.9995"), 3) - round(Decimal("9.9975"), 3), 3)
Decimal('0.002')

>>> round(round(Decimal("17.9995"), 3) - round(Decimal("17.9975"), 3), 3)
Decimal('0.002')
```

Also, as mentioned before, the decimal module exposes all parts of the standard,
for example, the rounding strategy can be modified from the standard
"Round Half Even". In order to do that you can use `Decimal.quantize()` method:

```python
>>> from decimal import ROUND_UP

>>> Decimal("4.5").quantize(Decimal("0"), rounding=ROUND_UP)
Decimal('5')
```

Decimal's rounding strategies include: `ROUND_CEILING`, `ROUND_DOWN`,
`ROUND_FLOOR`, `ROUND_HALF_DOWN`, `ROUND_HALF_EVEN`, `ROUND_HALF_UP`,
`ROUND_UP`, and `ROUND_05UP`.

Note that `quantize()` first argument is a `Decimal` object, that's to define
the exponent (which is extracted from the `Decimal` instance), personally I wish
there was a way to pass the exponent as a number, to be able to write more
concise code (or for `Decimal` to implement a `round()` method similar to the
built-in).


### Helping with significance eater

Onto more complex examples, we can rewrite the "Significance Eater" code seen in
[example]({{< ref "/blog/taming-floating-points.md/#the-significance-eater" >}} "The significance eater")
using `Decimal` to mitigate the behaviour:

```python
def significance_eater():
    x = Decimal("10.0") / Decimal("9.0")
    for _ in range(25):
        print(x)
        x = (x - Decimal("1.0")) * Decimal("10.0")
```

Running the above code...

```python
>>> significance_eater()
1.111111111111111111111111111
1.111111111111111111111111110
1.111111111111111111111111100
1.111111111111111111111111000
1.111111111111111111111110000
1.111111111111111111111100000
1.111111111111111111111000000
1.111111111111111111110000000
1.111111111111111111100000000
1.111111111111111111000000000
1.111111111111111110000000000
1.111111111111111100000000000
1.111111111111111000000000000
1.111111111111110000000000000
1.111111111111100000000000000
1.111111111111000000000000000
1.111111111110000000000000000
1.111111111100000000000000000
1.111111111000000000000000000
1.111111110000000000000000000
1.111111100000000000000000000
1.111111000000000000000000000
1.111110000000000000000000000
1.111100000000000000000000000
1.111000000000000000000000000
```

Note that the loss of significance is still happening due to the finite number
of digit stored by `Decimal` which are defined by decimal module precision
setting; however the catastrophic cancellation is a bit more manageable.

As mentioned before, decimal precision can be modified by the user, and it's
part of decimal `Context` object which is a singleton instance storing decimal
module's settings and events occurred since module import.

The `Context` instance can be accessed via `decimal.getcontext()` function:

```python
>>> from decimal import getcontext

>>> getcontext()
Context(prec=28, rounding=ROUND_HALF_EVEN, Emin=-999999, Emax=999999, capitals=1, clamp=0, flags=[], traps=[InvalidOperation, DivisionByZero, Overflow])
```

To modify the precision, simply:

```python
>>> getcontext().prec = 64

>>> Decimal("10.0") / Decimal("9.0")
Decimal('1.111111111111111111111111111111111111111111111111111111111111111')

>>> getcontext().prec = 30

>>> Decimal("10.0") / Decimal("9.0")
Decimal('1.11111111111111111111111111111')
```

That's nice!

The `Context` object can be used to modify other `decimal` module settings;
the rounding strategy, for instance, can be set to any of the rounding
[modes](https://docs.python.org/3/library/decimal.html#rounding-modes)
available in order to modify the behaviour of all rounding operations performed
by the module.

Another powerful tool are [Signals](https://docs.python.org/3/library/decimal.html#signals)
which can be set as `traps` in order to raise an exception as soon as an
operation triggers one, or can be monitored in the `flags` list.
See [Context](https://docs.python.org/3/library/decimal.html#decimal.Context)
full docs for more info.


### Helping with Simple Variance Calculator example

Let's see how decimal module can fix our "Simple Variance Calculator"
[example]({{< ref "/blog/taming-floating-points.md/#simple-variance-calculator" >}} "Simple Variance Calculator").

The definition of the function was:

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

In this case, we simply pass `Decimal` objects instead of `float`'s:

```python
>>> simple_variance([Decimal(str(x)) for x in np.random.uniform(1_000_000, 1_000_000.06, 100_000)])
Real variance: 0.00030060149786485578863919
Simp variance: 0.00030060148
```

The result is better but not quite there yet. Luckily we can increase the
precision to be able to get a more precise result:


```python
>>> getcontext().prec = 48

>>> simple_variance([Decimal(str(x)) for x in np.random.uniform(1_000_000, 1_000_000.06, 100_000)])
Real variance: 0.000301466911246091199506375296
Simp variance: 0.000301466911246091199506375296
```

Perfect!


### Fraction module

Another very interesting tool present in Python's standard library is the
[fractions](https://docs.python.org/3/library/fractions.html) module which can
greatly help in dealing with operations having inherently rational numbers.
Similarly to `decimal`, `fractions` implements a `Fraction` class which can be
used to represent rational numbers and implements rational arithmetic.

For example, to represent the fraction 11/10 using floats we'd use 1.1 but
that would result in an approximation:

```python
>>> from fractions import Fraction

>>> Fraction(1.1)
Fraction(2476979795053773, 2251799813685248)

>>> Fraction(1.1) == Fraction("11/10")
False
```

With `Fraction` we can exactly represent 11/10, and we can adjust floats to
their nearest rational number with its `limit_denominator()` method:

```python
>>> Fraction(1.1).limit_denominator() == Fraction("11/10")
True

>>> from math import pi, cos

>>> Fraction(cos(pi/3))
Fraction(4503599627370497, 9007199254740992)

>>> Fraction(cos(pi/3)).limit_denominator()
Fraction(1, 2)
```

Nice! More examples:

```python
>>> 0.1 + 0.1 + 0.1 == 0.3
False

>>> Fraction("1/10") + Fraction("1/10") + Fraction("1/10") == Fraction("3/10")
True

>>> float(Fraction("1/10"))
0.1

>>> float(Fraction("3/10"))
0.3

>>> Fraction(Decimal("1.1"))
Fraction(11, 10)

>>> Decimal("1.1").as_integer_ratio()
(11, 10)
```

Note how `Fraction` can be used to convert a rational number back to its nearest
float, and how a `Decimal` object represents 1.1 correctly to its ratio 11/10,
which can be obtained with `Decimal` method `as_integer_ratio()`.


Conclusions
-----------

The decimal module provides support for fast correctly rounded decimal
floating-point arithmetic, and it's great for debugging code with complex logic
involving `float`s.

In my opinion, overall helps a lot in writing more simple/readable code as it
integrates well with other Python’s built-in as well as other libraries like
pandas, although with some performance hit. Note: Arithmetic operations between
`Decimal` and `float` will raise `TypeError`.

Based on simple (by no mean exhaustive) tests, Decimal arithmetic operations
seems to be ~70% slower than float, worth noting that CPython uses libmpdec
where possible and other implementations of Decimal are available that might be
faster (e.g. Apache Arrow `pyarrow.decimal128`).

Lastly, `fractions` module is an elegant solution for handling rational numbers,
and its `limit_denominator()` method is very useful.

This concludes our two part journey in the vast and perilous world of
floating-point arithmetic. Hopefully these posts will help you avoid some of
the pitfalls and/or help in troubleshooting them.
