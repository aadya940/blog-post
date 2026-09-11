### How to fit a Smooth Curve? <i> Using SciPy </i>

A lot of things around us we take for granted are Smooth Curves, for example,
- <i> This roller coaster is a smooth path </i>

<img width="497" height="481" alt="image" src="https://github.com/user-attachments/assets/d589dc5a-3801-44cf-8e2e-89d311a14260" /> <br>

- <i> Your browser has paths defined for the beautiful fonts it renders such as </i>

<img width="497" height="481" alt="image" src="https://github.com/user-attachments/assets/02f862a3-43f5-4280-9f33-06a67dbc3ab2" /> <br>

- <i> Gradient Maps: These are a special type of smooth paths where the input is luminosity (instead of time) and the output is in color space </i>

<img width="497" height="481" alt="image" src="https://github.com/user-attachments/assets/09e87bac-d84d-46ff-8d24-b44916343fb5" /> <br>

These are smooth because their derivatives are continuous. The smoother the curve, the higher the order up to which its derivatives stay continuous.
For example, we know intuitively that this is smooth:

$$
y = \sin(x)
$$

But mathematically you can differentiate $y = \sin(x)$ an infinite number of times and still get a continuous curve. More importantly, if the
derivatives were discontinuous, the path would have sharp edges and applications like the roller coaster ride wouldn't be suitable. But smooth is under-
rated, mostly we're always given examples of motion for mathematical smoothness but it plays a huge role in aesthetics. There was a popular
artist, M.C. Escher, who built a career out of smoothness, here is some of his artwork:

<img width="497" height="600" alt="image" src="https://github.com/user-attachments/assets/ea0e2f19-ff60-4734-8058-bd4d58981fa5" /> <br> <br>
<img width="497" height="481" alt="image" src="https://github.com/user-attachments/assets/74de746d-2197-48ee-ba4f-7d9e684915fb" />

The mathematics of this was in fact later worked out in [this](https://pub.math.leidenuniv.nl/~smitbde/papers/2003-de_smit-lenstra-escher.pdf) paper.
So hopefully, I convinced you smooth curves are interesting.

### But, how do you fit a smooth curve?

Notice, the term "fit". We can fit a smooth curve on a bunch of data points <i>(xi, yi)</i> by minimizing this objective function:

<img width="582" height="95" alt="image" src="https://github.com/user-attachments/assets/749d8993-e0b1-46f0-b20d-8cfc4141330e" />

Here, $f(x)$ is the fitted curve, $f''$ is the second derivative of the same curve which measures how curvy or jumpy the curve
is. Hence, the first term in $J$ is how closely the curve fits the data, the second term is how jumpy/curvy the curve is and
$\lambda$ controls the amount of penalty to apply for bending.

When you optimise $J$ over the space of all smooth functions, you get a Spline. For a more detailed derivation, consult [this](https://github.com/aadya940/scipy-bspline-testing/blob/main/B_Splines_with_arbitary_knots-gcv.pdf)
paper.

### Splines

Splines are piecewise cubic polynomials. The domain of each piece is bounded by a pair of real numbers, let's call the sorted list of these
real numbers $t$.

It is a special type of piecewise polynomial function made by joining individual cubic (third-degree) polynomials together smoothly between a set of data points or knots. Instead of fitting a single, high-degree polynomial across an entire data set, which can cause wild oscillations known as Runge's phenomenon, splines use lower-degree cubic polynomials on smaller subintervals while maintaining continuous transitions from one piece to the next.
We're only assuming the cases that work on a <i>two-dimensional</i> plane for now.

So you can fit a smooth curve through a bunch of points using a spline and use $\lambda$ to control the amount of smoothing. With $\lambda = 0$, the curve interpolates the points, with $\lambda = \infty$ you get a straight line. Using this concept, you can start your artwork. Note that an image is just a bunch of horizontal or vertical lines stacked in order. So you can fit splines with separate $\lambda$'s and create images with focus on one particular object. Here is an example,

<img width="700" height="600" alt="image" src="https://github.com/user-attachments/assets/43241dba-70c5-4797-a52a-e4a83fe511da" /> <br>

Or you can use Splines, as a paintbrush, here's another example:

<img width="700" height="600" alt="image" src="https://github.com/user-attachments/assets/0a0e686d-5f9f-47ce-add9-77ca2ca2ac13" /> <br>

These have been built using <b>scipy.interpolate.make_smoothing_spline</b> function provided by the SciPy project which I had the pleasure to work on as an Intern (Summer, 2026) at Quansight Labs under Evgeni Burovski and Gagandeep Singh. Huge Shoutout to them!

### SciPy

The SciPy library provides a number of routines to work with different kinds of splines. The spline we discussed is provided by [scipy.interpolate.make_smoothing_spline](https://docs.scipy.org/doc/scipy/reference/generated/scipy.interpolate.make_smoothing_spline.html), it takes in the $x$, $y$ and $\lambda$ values and returns $f$, the fitted spline. But wait, what about the domain of the pieces of the piecewise cubic spline, which we named $t$, where's that?

It was internally assumed that $t$, the knots, are always equal to $x$, the data sites, so it was not in the API. This is a popular choice since there is a classical theorem that when you set $t$ = $x$ you always get the curve that best minimizes the objective function we discussed earlier. Internally, SciPy builds matrices to solve this. The size of the system depends on the number of pieces in the piecewise function, and with $t$ = $x$ the number of pieces grows with the data. For `1,000,000` datapoints, the size of the matrix is `1,000,000 x 1,000,000`. This is the primary tradeoff of the knots at data points choice. Hence, some people may want to pass a smaller $t$ to `make_smoothing_spline` to get a slightly worse curve but much faster. This was not available in SciPy yet, I worked through the summer to add it. On the good side, these matrices are often banded (<i>Because Piecewise, Only a few diagonals are non-zero </i>) with band size `k + 1` where `k` is the degree of each piece. Here, since $k$ is just $3$, the number of non-zero elements is $(k + 1) * (num. datapoints)$, in this case it's $4,000,000$. By the way, the matrix I'm talking about is called the "design matrix", It is the $X$ when solving the equation $y = X * c$ where $c$ are the coefficients that scale the influence of each column of the design matrix.

Expanding it further, in our case we're solving this particular equation:

<img width="500" height="78" alt="image" src="https://github.com/user-attachments/assets/941d2b26-5239-4f77-bb65-0223cb78c0b8" />

Here, the new $\Omega$ is just the matrix form of the integral of the squared second derivative we saw earlier. <br> Now we have two options, either differentiate and integrate numerically every time to compute $\Omega$ or derive a general matrix form of $\Omega$ which circumvents this procedure, and once you have matrices with nice properties you can apply optimizations on them, inspect them etc. <br> So what is the matrix form and how to compute it? <br> That was about half of my internship. Deriving $\Omega$ using papers and books going back to the 1980's. Other implementations like the R programming language's libraries are GPL licensed, so we deliberately did not look at their source code (only used their numerical output as a black-box check). Apart from that, most of MATLAB, Octave, Julia etc. don't support user defined knot vectors either.

SciPy did use matrices but they assume the $t$ = $x$ thing and hence, their matrices are more specialized than what the general case needs. The complete derivation is available in [this](https://github.com/aadya940/scipy-bspline-testing/blob/main/B_Splines_with_arbitary_knots-gcv.pdf) paper. Also, when $t$ = $x$ the design matrix was a square matrix, now it is not, so we can't apply some shortcuts we applied earlier. Furthermore, previously in SciPy you could pass $\lambda$ = None and SciPy would automatically find the appropriate one using an algorithm called Generalized Cross Validation (GCV). This algorithm assumed $t$ = $x$ as well, so I went on to derive the general case for that too. I spent some parts of the Internship on Optimizations, for example, whenever you find that a matrix is a special type of matrix called a [Hermitian matrix](https://en.wikipedia.org/wiki/Hermitian_matrix) (symmetric, in our real-valued case) and positive definite, you can use SciPy's `solveh_banded` solver which is several times faster than the general `solve_banded` solver, or whenever you have a banded matrix (<i> which is nearly always in this case </i>), use SciPy's [diags_array](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.diags_array.html) for optimized routines, furthermore the [LAPACK](https://www.netlib.org/lapack/) FORTRAN library which SciPy provides Python interfaces to has several banded linear algebra routines. I also derived in the paper mentioned above, the relation between the [condition number](https://en.wikipedia.org/wiki/Condition_number) and the smoothing parameter $\lambda$, this helps us understand the domains where the algorithm is numerically stable, where it is not, where it is unsolvable and why. We went through several code reviews during these parts.

### What do users get now?
A new Keyword Argument:

```python
from scipy.interpolate import make_smoothing_spline

# before: knots are fixed at the data sites
spl = make_smoothing_spline(x, y, lam=lam)

# now: bring your own knots
spl = make_smoothing_spline(x, y, lam=lam, t=my_knots)

# or let GCV pick lambda on your knots, that works too
spl = make_smoothing_spline(x, y, t=my_knots)
```

What this buys you:
- <b> The solve is sized by your knots, not your data. </b> Remember the `1,000,000 x 1,000,000` matrix from before? Pass 100 knots and the system is roughly `100 x 100` (banded, on top of that), no matter how many points you have. The data only enters through small products like $X^T y$.
- <b> You decide where the flexibility goes. </b> Knots are where the curve is allowed to bend. Put more of them where your data actually does something and few where it is quiet. Every image in this post uses that control.
- <b> $\lambda$ = None still works. </b> The GCV search was re-derived for arbitrary knots, so automatic smoothing is not a $t = x$ exclusive anymore.
- <b> Nothing changes if you don't opt in. </b> Leave $t$ out and you get exactly the old behavior. In fact the test suite demands it: with knots at the data and the same $\lambda$, the new path reproduces the old one to about `1e-12`, and the penalty matrix $\Omega$ is computed in closed form, no quadrature error to budget for.

The feature lives in [scipy/scipy#25862](https://github.com/scipy/scipy/pull/25862) and the GCV follow-up is in review, so it makes its way to a SciPy release near you soon.

#### Tutorial 1: Lots of points, few knots
Let's fit 200,000 noisy samples of a smooth signal using just 12 interior knots:

```python
>>> import numpy as np
>>> from scipy.interpolate import make_smoothing_spline

>>> rng = np.random.default_rng(42)
>>> x = np.sort(rng.uniform(0, 1, 200_000))
>>> y = 0.4 + 0.3*np.sin(5.1*x) + 0.2*np.sin(13.7*x + 0.8) \
...  + rng.normal(0, 0.25, x.size)
>>>
>>> t = np.linspace(0, 1, 14)[1:-1]        # 12 interior knots
>>> spl = make_smoothing_spline(x, y, lam=1e-7, t=t)
```

<br><br>

<img width="1590" height="510" alt="image" src="https://github.com/user-attachments/assets/70ddfecf-7191-4c10-b0ba-c1de2bc6855c" />

<br><br>

The noise here has amplitude `0.25` and the fitted curve is within `0.007` of the
true signal. Now recall the matrix sizes from earlier. With `t = x` this would be a
`200,000 x 200,000` system. With 12 interior knots it is `16 x 16`, and banded. The
data only enters through small products like $X^T y$, so even if you add more
points, the system to solve stays the same size.

#### Tutorial 2: Put the knots where the data bends
Since knots decide where the curve can flex, where you place them matters. Here is
a signal which is mostly quiet except a sharp bump at `x = 0.7`. We fit it twice,
both times with exactly 8 interior knots:

```python
>>> t_uniform = np.linspace(0, 1, 10)[1:-1]
>>> t_placed  = np.array([0.25, 0.5, 0.62, 0.66, 0.70, 0.74, 0.78, 0.9])
>>>
>>> spl_u = make_smoothing_spline(x, y, lam=1e-9, t=t_uniform)
>>> spl_p = make_smoothing_spline(x, y, lam=1e-9, t=t_placed)
```

<br><br>

<img width="1590" height="510" alt="image" src="https://github.com/user-attachments/assets/737dd50d-ea72-4d84-920d-1503d353d3b0" />

<br><br>

Same data, same $\lambda$, same number of coefficients. The uniform knots
oversmooth the bump and wiggle around in the quiet region, RMSE against the truth
is `0.053`. The placed knots are clustered around the bump, so the bump comes out
clean and the rest stays calm, RMSE `0.013`. The small tick marks in the plots show
where the knots are.

#### Tutorial 3: Don't pick lambda at all
If picking knots by hand felt like work, picking $\lambda$ is worse, nobody has
intuition for its units. So leave it out and GCV picks it for you, now on your
knots too:

```python
>>> spl = make_smoothing_spline(x, y, t=t)    # no lam
```

<br><br>
<img width="1590" height="510" alt="image" src="https://github.com/user-attachments/assets/5617d2e2-993b-4c4d-bd28-6ee28b2c6343" />
<br><br>

On the left is what happens internally. GCV asks, for each candidate $\lambda$,
how well would the fit predict each point if that point were left out, and picks
the minimum. Note the x-axis, it's $\log_{10}(\lambda / r)$, not $\log_{10}(\lambda)$.
That ratio $r$ is what makes the search work in any unit system. $\lambda$ has units,
so if you rescale $x$, the right $\lambda$ changes too. You can check this: I fit
the same measurements with $x$ in metres and in kilometres, and GCV chose
`2.40e-03` and `2.40e-12`, a factor of $10^9$ apart, exactly the $(10^3)^3$ the
units demand, and the two fitted curves are the same curve. The search is over
the dimensionless variable, so the fixed window works for everyone. This also
addresses a long-standing SciPy issue where the old search failed on data
with large or small $x$ scales.


### References

- Image 1: https://viterbi-web.usc.edu/~jbarbic/cs420-s17/assignments/assign2/assign2.html
- Image 2: https://www.theguardian.com/artanddesign/2015/jun/20/the-impossible-world-of-mc-escher
- Image 3: Print Gallery by M.C. Escher

