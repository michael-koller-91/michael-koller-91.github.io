---
layout: default
title: "Sliding window Hilbert transform"
date: 2023-10-09
---

# A derivation

(If equations aren't displayed properly, please reload the page.)

We are given a vector $x = (x_0, x_1, \dots, x_{L-1})$ of $L$ time samples.
For every subvector $s_t = (x_t, x_{t+1}, \dots, x_{t+N-1})$ of $N$ consecutive samples of $x$,
we would like to compute the [Hilbert transform](https://en.wikipedia.org/wiki/Hilbert_transform) $H(s_t)$.

In principle, we could do the following in Python:
```python
import numpy as np
from scipy.signal import hilbert

def sliding_window_hilbert_v1(x, N):
    L = len(x)
    H = np.zeros((L - N + 1, N), dtype=complex)
    for t in range(L - N + 1):
        # x[t:t + N] is the subvector s_t
        H[t, :] = hilbert(x[t:t + N])
    # row t of H contains the Hilbert transform of the subvector s_t
    return H
```
This is an expensive solution because we compute the Hilbert transform
$L - N + 1$ times without taking previous results into account.

In this article, we develop a version of this function which calls `hilbert` only once
and then computes the Hilbert transform of row $t+1$ based on the Hilbert transform of row $t$.

## Hilbert transform
The Hilbert transform of $s_t$ can be computed using a Fourier transform,
followed by windowing, followed by an inverse Fourier transform.
Writing $v[n]$ for the element $n+1$ of some vector $v$ (as it is done in Python for example),
the Fourier transform $F_t$ of the subvector $s_t$ can be computed as follows

$$\begin{equation}
    F_t[n] = \sum_{d=0}^{N-1} s_t[d] e^{-\frac{j2\pi}{N}dn} 
    = \sum_{d=0}^{N-1} x_{t+d} e^{-\frac{j2\pi}{N}dn} 
\end{equation}$$

for $n = 0, 1, \dots, N - 1$.
Note, since $s_t$ is a subvector of $x$, we have $s_t[d] = x_{t+d}$.

We define a length-$N$ filter vector $W$ as

$$\begin{equation}
    W[n] = \begin{cases}
        1 & \text{for } n = 0, \frac{N}{2}\\
        2 & \text{for } n = 1, \dots, \frac{N}{2} - 1\\
        0 & \text{for } n = \frac{N}{2} + 1, \dots, N - 1
    \end{cases}
\end{equation}$$

and can now compute the Hilbert transform of $s_t$:

$$\begin{equation}
    H(s_t)[d] = \frac{1}{N} \sum_{n=0}^{N-1} W[n] F_t[n] e^{\frac{j2\pi}{N}dn}
\end{equation}$$

for $d = 0, \dots, N - 1$.
This last equation is the inverse Fourier transform of the elementwise product of $W$ and $F_t$.

## Sliding Hilbert transform

Instead of performing both a Fourier transform and an inverse Fourier transform for every subvector $s_t$, we want to make use of $H(s_t)$ when we compute the next subvector's Hilbert transform $H(s_{t+1})$.
To this end, we write down the formula for $H(s_{t+1})[d]$ and plug in the formula for $F_{t+1}[n]$:

$$\begin{aligned}
    H(s_{t+1})[d]
    &= \frac{1}{N} \sum_{n=0}^{N-1} W[n] F_{t+1}[n] e^{\frac{j2\pi}{N}dn}
    \\&= \frac{1}{N} \sum_{n=0}^{N-1} W[n] \Bigg(
        \sum_{k=0}^{N-1} s_{t+1}[k] e^{-\frac{j2\pi}{N}kn}
    \Bigg) e^{\frac{j2\pi}{N}dn}
    \\&= \frac{1}{N} \sum_{n=0}^{N-1} W[n] \Bigg(
        \sum_{k=0}^{N-1} x_{t+1+k} e^{-\frac{j2\pi}{N}kn}
    \Bigg) e^{\frac{j2\pi}{N}dn}.
\end{aligned}$$

The goal is now to massage this expression until $H(s_t)$ appears.
In what follows, we make use of $e^{-j2\pi n} = 1$ for all integers $n$.

$$\begin{aligned}
    H(s_{t+1})[d]
    &= \frac{1}{N} \sum_{n=0}^{N-1} W[n] \Bigg(
        \sum_{k=0}^{N-1} x_{t+1+k} e^{-\frac{j2\pi}{N}kn}
    \Bigg) e^{\frac{j2\pi}{N}dn} 
    \\&= \frac{1}{N} \sum_{n=0}^{N-1} W[n] \Bigg(
        \sum_{k=1}^N x_{t+k} e^{-\frac{j2\pi}{N}(k-1)n}
    \Bigg) e^{\frac{j2\pi}{N}dn}
    \\&= \frac{1}{N} \sum_{n=0}^{N-1} W[n] \Bigg(
        \sum_{k=1}^N x_{t+k} e^{-\frac{j2\pi}{N}kn}
    \Bigg) e^{\frac{j2\pi}{N}n} e^{\frac{j2\pi}{N}dn}
    \\&= \frac{1}{N} \sum_{n=0}^{N-1} W[n] \Bigg(
        x_{t+N} e^{-j2\pi n} + \sum_{k=1}^{N-1} x_{t+k} e^{-\frac{j2\pi}{N}kn}
    \Bigg) e^{\frac{j2\pi}{N}(d+1)n}
    \\&= \frac{1}{N} \sum_{n=0}^{N-1} W[n] \Bigg(
        x_{t+N} + F_t[n] - x_t
    \Bigg) e^{\frac{j2\pi}{N}(d+1)n}
    \\&= \frac{1}{N} \sum_{n=0}^{N-1} W[n] F_t[n] e^{\frac{j2\pi}{N}(d+1)n}
        + \frac{1}{N} \sum_{n=0}^{N-1} W[n]
        (x_{t+N} - x_t) e^{\frac{j2\pi}{N}(d+1)n}
    \\&= H(s_t)[d+1] + \big(x_{t+N} - x_t\big)
    \underbrace{\frac{1}{N} \sum_{n=0}^{N-1} W[n] e^{\frac{j2\pi}{N}(d+1)n}}_{=:w[d]}.
\end{aligned}$$

We have achieved our goal of making $H(s_t)$ appear.
Two comments are in order here.

First, the last element $H(s_{t+1})[N - 1]$ of $H(s_{t+1})$ depends on $H(s_t)[N]$
which strictly speaking does not exist since $H(s_t)$ is a length-$N$ vector.
However, if we look at the definition of $H(s_t)[d]$ above,
we can see that $H(s_t)[N] = H(s_t)[0]$ holds due to the periodic nature of the exponential function.

Second, the factor $w[d] = \frac{1}{N} \sum_{n=0}^{N-1} W[n] e^{\frac{j2\pi}{N}(d+1)n}$ seems to be expensive to compute.
However, since it does not depend on the data samples $x$, $w[d]$ can easily be precomputed for $d = 0, 1, \dots, N - 1$.
Interestingly, $w$ is a time-shifted version of the inverse Fourier transform of $W$.

In summary, the Hilbert transform $H(s_{t+1})$ is a time-shifted version of $H(s_t)$ plus a correction term which replaces the oldest data sample $x_t$ with the newest data sample $x_{t+N}$:

$$\begin{equation}
    H(s_{t+1})[d] = H(s_t)[d+1] + \big(x_{t+N} - x_t\big) w[d]
\end{equation}$$

If $H(s_t)$ is given, computing $H(s_{t+1})$ costs only $N$ multiplications and $N + 1$ summations.

## Sliding Hilbert transform in Python

We can readily implement our derived function:
```python
def sliding_window_hilbert_v2(x, N):
    L = len(x)

    W = np.zeros(N)
    if N % 2 == 0:
        W[0] = 1
        W[N // 2] = 1
        W[1 : N // 2] = 2
    else:
        W[0] = 1
        W[1 : (N + 1) // 2] = 2
    # time-shifted inverse Fourier transform of W
    w = np.roll(np.fft.ifft(W), -1)

    H = np.zeros((L - N + 1, N), dtype=complex)
    for t in range(L - N + 1):
        if t== 0:
            # H(s_0) needs to be computed explicitly
            H[t, :] = hilbert(x[:N])
        else:
            for d in range(N):
                # the time shift H(s_t)[d] = H(s_{t-1})[d+1]
                if d == N - 1:
                    H[t, d] = H[t - 1, 0]
                else:
                    H[t, d] = H[t - 1, d + 1]
                # the correction term
                H[t, d] += (x[t - 1 + N] - x[t - 1]) * w[d]
    # row t of H contains the Hilbert transform of the subvector s_t
    return H
```

Note, since for loops are very slow in Python,
it makes sense to just-in-time-compile this function using `numba` for example.

[back](./../)
