---
layout: default
title: "How to print floating-point numbers: A review of the Dragon 2 algorithm"
date: 2023-03-26
---

# How to print floating-point numbers:<br />A review of the Dragon 2 algorithm

Decimal numbers like `0.21` do not have an exact representation by means of a finite-length binary number.
In Python, we can see the best approximation of `0.21` by means of a double-precision floating-point number (a `double`) by printing `0.21` with large precision:
```python
>>> nr = 0.21
>>> print(f'{nr:.100}')
0.2099999999999999922284388276239042170345783233642578125
```
We see a number with 55 digits after the decimal point which is clearly not equal to `0.21`.
Yet, we have the following:
```python
>>> nr
0.21
>>> print(nr)
0.21
```
That is, if we don't specify a precision and just print `nr`, we get the expected `0.21`
which we asigned to `nr` in the beginning.
Even more interestingly, we have:
```python
>>> 0.2099999999999999922284388276239042170345783233642578125
0.21
```
Thus, there runs an algorithm in the background 
which prints decimals with a long fractional part as a more easily readable string.
In this article, we take a look at one such algorithm: Dragon 2.

The algorithm described here is from the paper
[How to Print Floating-Point Numbers Accurately by G. Steele Jr. and J. White](https://lists.nongnu.org/archive/html/gcl-devel/2012-10/pdfkieTlklRzN.pdf).

## Dragon 2

The first algorithm to print floating-point numbers that is described in the mentioned paper is called _Dragon 2_.
The input of Dragon 2 is a floating-point number $v = m \cdot 2^{e - p}$
where $m$, $e$, and $p$ are integers such that $p \geq 0$ and $0 < m < 2^p$.
The algorithm then prints a string of the form

$$k.F \cdot 10^x$$

where $k$, $F$, and $x$ are integers and where the decimal point is printed between $k$ and $F$.

---

**Examples:**
Consider $m = 821$ and $p = 10$.
Let's choose $e = 3$
which yields $v = 821 \cdot 2^{3 - 10} = 6.4140625$.
Dragon 2 prints this as $6.414 \cdot 10^0$.
For another example, let's choose $e = 14$
which yields $v = 821 \cdot 2^{14 - 10} = 13136$.
Dragon 2 prints this as $131.4 \cdot 10^2$.

---

Before we describe the Dragon 2 algorithm,
we collect some facts about floating-point numbers.
Evidently, we need $p$ bits to store the integer $m$.
We can write $m$ as the binary number $m_1 m_2 \dots m_p$.
Somewhere between these $p$ bits or before bit $m_1$ or after bit $m_p$ is a binary point.
With $e - p > 0$, we can move the binary point $e - p$ bits to the right.
With $e - p < 0$, we can move the binary point $e - p$ bits to the left.
It is up to the standard to decide where this binary point is when $e - p = 0$ holds.
For instance, the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard puts it right before $m_1$.

It is important to note the following.
The number of significant bits of $v$ is given by $p$.
This does not depend on where the binary point is
and it does not depend on the range of values we allow the exponent $e$ to take.
If we want to express a number more precisely, we need to increase $p$.
This is obvious if we take a look at the standard scientific notation in the decimal system.
If we are only allowed to use $p = 3$ digits $d_1$, $d_2$, and $d_3$ to write a number of the form
$d_1 . d_2 d_3 \cdot 10^{e-3}$,
then it doesn't matter how we choose $e$ – we will always only have $p = 3$ digits precision.

The idea of Dragon 2 is now this.
We find an integer $x$ such that $v' = v \cdot 10^{-x}$ fulfills $0 < v' < 2^p$.
The number $v'$ has an integer part $k$ (the part before the decimal point)
and a fractional part $f$ (the part after the decimal point).
For instance, the integer part of $21.16$ is $k = 21$ and the fractional part is $f = 0.16$.
Representing the integer $k$ as a binary number would require $\mathrm{ceil}(\log_2(k))$ bits (the function [ceil](https://en.wikipedia.org/wiki/Floor_and_ceiling_functions) rounds up).
In other words, the integer part $k$ is worth $\mathrm{ceil}(\log_2(k))$ bits.
Due to $v' < 2^p$, we know that $\mathrm{ceil}(\log_2(k)) \leq p$ holds.
Thus, we have enough precision for the integer part $k$.
Printing the integer part $k$ consumes $\mathrm{ceil}(\log_2(k))$ bits precision.
The amount of precision that is left, namely $n = p - \mathrm{ceil}(\log_2(k))$,
can be spent on printing the fractional part.
We explain how to print a fractional part with a given precision in the next section.
For now, let's assume we have such an algorithm.

We can then summarize Dragon 2 in the following steps:
1. find $x$ such that $v' = v \cdot 10^{-x} < 2^p$ holds
2. print the integer part $k$ of $v'$
3. print a decimal point
4. compute the remaining precision $n = p - \mathrm{ceil}(\log_2(k))$
5. print the fractional part with a precision of $n$ (this prints the integer $F$)
6. print: $\cdot 10$
7. print $x$ as an exponent

The magic happens in step five.
We discuss how this is done in the next section.

## How to print fixed-point numbers accurately

In step five of Dragon 2, we face the problem of printing a fractional part
with a precision of $n$.
We can alternatively say that we want to print a number in the interval $[0, 1[$
that has an $n$ bits long fractional part.
Such a number has the form
$0.f_1 f_2 \dots f_n$ with bits $f_k \in \{ 0, 1 \}$.
This is also known as an $n$-bit fixed-point number.
Its decimal value is given by $\sum_{k=1}^n f_k \cdot 2^{-k}$.
For instance, the binary number $f = 0.01111101$ with $n = 8$ has the decimal value $f = 0.48828125$.

### Preliminaries

The difference between two consecutive $n$-bit fixed-point numbers is $2^{-n}$.
Thus, the maximal difference between any decimal number in $[0, 1[$ and the closest $n$-bit fixed-point number is $M_n = 2^{-n} / 2$.
As a consequence, if $g$ is an $n$-bit fixed-point number,
all decimal numbers in the interval $] g - M_n, g + M_n [$ are represented by the same number $g$.
Continuing the example from above, we would for instance have

$$\begin{equation}
    \Big] f - M_{8},\;\; f + M_{8} \Big[ \;\;=\;\; \Big] 0.486328125,\;\; 0.490234375 \Big[.
\end{equation}$$

Reading in any number from this interval results in the same binary number $f$.
Thus, the number which we want to print for $f$ should come from this interval
such that repeated printing and reading does not corrupt our data.
Among all numbers in $] f - M_8, f + M_8 [$,
the number $0.49$ is the shortest (in terms of digits after the decimal point),
which is why we consider $0.49$ to be the easiest-to-read representation of $f$.
The goal therefore is to find an algorithm which takes $f$ and $n$ as input
and prints $0.49$ as its output.

### Main idea of the fixed-point printing algorithm

**Goal:**
We give an $n$-bit fixed-point number $f \in [0, 1[$ and $n$ to the algorithm
and it prints a decimal number $0.F_1 F_2 \dots F_N$ with $F_{k} \in \{0, 1, \dots, 9\}$.
The number of digits, $N$, is determined by the algorithm and is as small as possible.

We develop the algorithm using the example $f = 0.48828125$ and $n = 8$.
We want to generate one digit after the other for as long as it is needed.
In the first step, $k = 1$, we generate the first digit $F_1$
and if we were to terminate the algorithm after this step, we would print the number $0.F_1$.
If the termination criterion is not met, we continue to step $k = 2$ where we generate a new digit $F_2$ and so on.

In our example, there are two valid options for $F_1$:
either $4$ or $5$ because $0.4$ and $0.5$ are the two numbers of the form $0.F_1$ which are closest to $f$.
We set $F_1 = 4$ and write the two candidates as $0.F_1 = 0.4$ and $0.F_1 + 0.1 = 0.5$
Note that $0.1$ is the difference between any two consecutive numbers of the form $0.F_1$.

We now need to check if one of the two candidates lies in the interval around $f$ (the interval in Equation (1)).
Since we have $0.F_1 \leq f$, we can check if $0.F_1$ lies in the interval by checking
if $f - 0.F_1 < M_8$ holds.
Similarly, since we have $0.F_1 + 0.1 \geq f$, we can check if the second candidate $0.F_1 + 0.1$ lies in the interval by checking
if $0.F_1 + 0.1 - f < M_8$ holds.
So, in step $k = 1$ of our algorithm, we check if

$$\begin{aligned}
    &f - 0.F_1 < M_8 \quad\text{or}\quad 0.F_1 + 0.1 -f < M_8 \\
    \Leftrightarrow\quad & f - 0.F_1 < M_8 \quad\text{or}\quad f - 0.F_1 > 0.1 - M_8
\end{aligned}$$

holds.
We can now see why it was convenient to express the second candidate in the way we did it:
We only need to compute one remainder $R_1 = f - 0.F_1$ and can use it in two comparisons.

Using $M_8 = 0.001953125$ and $0.1 - M_8 = 0.098046875$,
step $k = 1$ of our algorithm looks as follows:

$$k = 1: F_1 = 4, \quad R_1 = f - 0.F_1 = 0.08828125, \quad R_1 > M_8, \quad R_1 < 0.1 - M_8.$$

We conclude that neither $0.F_1 = 0.4$ nor $0.F_1 + 0.1 = 0.5$ lies in the interval
around $f$ and we go to step $k = 2$.

Note that if the larger candidate $0.F_1 + 0.1$ does not lie in the interval,
then appending further digits to it will only increase this number and thus move it farther away from the interval.
For this reason, we continue with $F_1 = 4$ and the smaller candidate $0.F_1$
and append a new digit to it in step $k = 2$ to make it bigger and thereby move it closer to the interval.
 
In step $k = 2$, $F_1$ is fixed and we want to generate a new digit $F_2$, yielding the number $0.F_1 F_2 = 0.4F_2$.
Since we have $f = 0.48828125$,
the two closest numbers of the form $0.4F_2$ are $0.48$ and $0.49$.
Similar to before, we set $F_2 = 8$ and express the two candidates as $0.4F_2 = 0.48$ and $0.4F_2 + 0.01 = 0.49$.
Now, $0.01$ is the difference between any two consecutive numbers of the form $0.4F_2$.
Again, we want to know if any of the two candidates lies in the interval around $f$.
This is achieved by comparing $0.4F_2$ and $0.4F_2 + 0.01$ to $M_8$.
So, step $k = 2$ of our algorithm looks as follows:

$$k = 2: F_2 = 8, \quad R_2 = f - 0.4F_2 = 0.00828125, \quad R_2 > M_8, \quad R_2 > 0.01 - M_8$$

with $M_8 = 0.001953125$ and $0.01 - M_8 = 0.008046875$.
While $R_2 > M_8$ indicates that the first candidate is still not in the interval around $f$,
the inequality $R_2 > 0.01 - M_8$ tells us that the second candidate does lie in the interval.
By construction, it is also the shortest number in this interval.
Thus, we stop iterating and print the second candidate $0.4F_2 + 0.01 = 0.49$.

### The general rule

In general, in step $k$, we set $F_k$ equal to the $k$-th fractional digit of $f$
and the two candidates are given by
$0.F_1 F_2 \dots F_k$ and $0.F_1 F_2 \dots F_k + 10^{-k}$.
Thus, we compute the remainder $R_k = f - 0.F_1 F_2 \dots F_k$ and check if
$R_k < M_n$ (candidate one lies in the interval around $f$) or
$R_k > 10^{-k} - M_n$ (candidate two lies in the interval around $f$) holds.
If one of the two conditions is fulfilled, we terminate and print the corresponding candidate.
If no condition is met, we continue with step $k + 1$.

### A simple Python implementation

Since on a computer we don't have access to the decimal representation of $f$,
we need to compute its digits instead of reading them off.
The $k$-th digit of $f$ can be computed as $F_k = \mathrm{floor}(f \cdot 10^k)$,
where [floor](https://en.wikipedia.org/wiki/Floor_and_ceiling_functions) rounds down to the nearest integer.

Note that the digits $k$, $k + 1$, $\dots$ of the remainder $R_{k-1} = f - 0.F_1 F_2 \dots F_{k-1}$
are equal to the digits $k$, $k + 1$, $\dots$ of $f$
because the corresponding digits of $0.F_1 F_2 \dots F_{k-1}$ are all $0$
(we could write $0.F_1 F_2 \dots F_{k-1} = 0.F_1 F_2 \dots F_{k-1} 0 0 0 0 0 \dots$).
Hence, we can also compute the $k$-th digit of $f$ as $F_k = \mathrm{floor}(R_{k-1} \cdot 10^k)$.

Furthermore, we might as well compute $\tilde{R}\_{k} = 10^k \cdot R_k$
and compare it to $10^k \cdot M_n$ and $10^k \cdot (10^{-k} - M_n) = 1 - 10^k \cdot M_n$
to find out if any of the two candidates lies in the interval around $f$.
This formulation has the advantage that we don't have to compute both $10^k$ as well as $10^{-k}$
in step $k$ but instead we only multiply the previous $\tilde{R}\_{k-1}$ by $10$
and we also multiply the previous $10^{k-1} \cdot M_n$ by $10$.
Then, a possible Python implementation of the whole algorithm might look as follows
(we make us of `math.floor`):

```python
def fixed_point_print(f, n):
    k = 0
    R = f   # this corresponds to the remainder R_0 = f - 0
    M = 2 ** -n / 2
    output = '0.'   # initialize the output string
    while True:
        k = k + 1
        R_x_10 = R * 10     # multiply the previous remainder by 10 in order to
        F = floor(R_x_10)   # generate the new digit F_k
        R = R_x_10 - F
        M = M * 10
        output = output + f'{F}'    # append a new digit, this corresponds to
                                    # the first candidate
        if (R < M) or (R > 1 - M):
            break
    if R >= 0.5:
        output = output[:-1] + f'{F + 1}'   # increment the last digit, i.e.,
                                            # choose the second candidate
    print(output)
```
As a test, we get
```python
>>> fixed_point_print(0.48828125, 8)
0.49
```

## Limitations of Dragon 2

The fixed-point printing algorithm
which is used in step five of Dragon 2 is accurate:
repeatedly printing and reading will always lead to the same binary number.
Dragon 2, however, is not entirely accurate.
This issue has two sources.

First, while the distance between two _fixed_-point numbers is constant,
the distance between two _floating_-point numbers is only constant in most cases.
That is to say, in most cases, if $v$ is a floating-point number,
the next larger floating-point number $v^+$ is as far away
as the next smaller floating-point number $v^-$.
However, if $v$ is an integral power of $2$,
the distance $v^+ - v$ to the next larger number is two times the distance
$v - v^-$ to the next smaller number.
This can lead to the problem that printing $v$ and then reading the result back in
does not yield $v$ again.

Second, in the first step of Dragon 2, we need to find $x$ such that $v' = v \cdot 10^{-x} < 2^p$ holds.
This scaling needs to be performed as a floating-point multiplication and it can happen that due to
rounding two different (very small) numbers get scaled to the same number $v'$.
As a result, both numbers get printed as the same number and reading this number back in
cannot possibly recover the two original numbers.

As a summary, as long as we do not need to read the printed numbers back in,
Dragon 2 is a good simple floating-point printing algorithm.
For applications where a lack of accuracy is not acceptable,
the authors of [How to Print Floating-Point Numbers Accurately](https://lists.nongnu.org/archive/html/gcl-devel/2012-10/pdfkieTlklRzN.pdf) also derive an algorithm (called _Dragon 4_)
which only performs integer arithmetic and is accurate for floating-point numbers.

The _Dragon_ naming convention is very popular.
For example, there are the two floating-point printing algorithms
_Grisu_ and _Ryu_ both of which refer to particular dragons and there is the github repository
[Drachennest](https://github.com/abolz/Drachennest) (German for _dragon nest_) which collects floating-point printing algorithms.
The linked paper as well as the github repository are good starting points for the reader
who would like to know more about the interesting printing topic.
