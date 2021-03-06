---
title: "Using RcppArmadillo to price European Put Options"
author: "Davis Vaughan and Dirk Eddelbuettel"
license: GPL (>= 2)
mathjax: true
tags: armadillo basics
summary: "Three different ways for computing Black-Scholes put option prices are discussed."
layout: post
src: 2018-02-28-black-scholes-three-ways.Rmd
---

### Introduction

In the quest for ever faster code, one generally begins exploring ways to integrate C++
with R using [Rcpp](http://www.rcpp.org). This post provides an example of multiple
implementations of a European Put Option pricer. The implementations are done in pure R,
pure [Rcpp](http://www.rcpp.org) using some [Rcpp](http://www.rcpp.org) sugar functions,
and then in [Rcpp](http://www.rcpp.org) using
[RcppArmadillo](http://dirk.eddelbuettel.com/code/rcpp.armadillo.html), which exposes the
incredibly powerful linear algebra library, [Armadillo](http://arma.sourceforge.net/).

Under the [Black-Scholes model](https://en.wikipedia.org/wiki/Black%E2%80%93Scholes_model) The value of a European put option has the closed form solution:

$$ V = K e^{-rt} N(-d_2) - S e^{-yt} N(-d_1) $$

where

$$ 
\begin{equation}
  \begin{aligned}
V    &= \text{Value of the option} \\
r    &= \text{Risk free rate} \\
y    &= \text{Dividend yield} \\
t    &= \text{Time to expiry} \\
S    &= \text{Current stock price} \\
K    &= \text{Strike price} \\
N(.) &= \text{Normal CDF}
  \end{aligned}
\end{equation}
$$

and

$$ 
\begin{equation}
  \begin{aligned}
d_1    &= \frac{log(\frac{S}{K}) + (r - y + \frac{1}{2} \sigma^2)t}{\sigma \sqrt{t}}  \\
d_2    &= d_1 - \sigma \sqrt{t}\\
  \end{aligned}
\end{equation}
$$

Armed with the formulas, we can create the pricer using just R.


{% highlight r %}
put_option_pricer <- function(s, k, r, y, t, sigma) {

    d1 <- (log(s / k) + (r - y + sigma^2 / 2) * t) / (sigma * sqrt(t))
    d2 <- d1 - sigma * sqrt(t)

    V <- pnorm(-d2) * k * exp(-r * t) - s * exp(-y * t) * pnorm(-d1)

    V
}

# Valuation with 1 stock price
put_option_pricer(s = 55, 60, .01, .02, 1, .05)
{% endhighlight %}



<pre class="output">
[1] 5.52021
</pre>



{% highlight r %}
# Valuation across multiple prices
put_option_pricer(s = 55:60, 60, .01, .02, 1, .05)
{% endhighlight %}



<pre class="output">
[1] 5.52021 4.58142 3.68485 2.85517 2.11883 1.49793
</pre>

Let's see what we can do with [Rcpp](http://www.rcpp.org). Besides explicitely stating the
types of the variables, not much has to change. We can even use the sugar function,
`Rcpp::pnorm()`, to keep the syntax as close to R as possible. Note how we are being
explicit about the symbols we import from the `Rcpp` namespace: the basic vector type, and
well the (vectorized) 'Rcpp Sugar' calls `log()` and `pnorm()` calls.  Similarly, we use
`sqrt()` and `exp()` for the calls on an atomic `double` variables from the C++ Standard
Library. With these declarations the code itself is essentially identical to the R code
(apart of course from requiring both static types and trailing semicolons).


{% highlight cpp %}
#include <Rcpp.h>
                                        
using Rcpp::NumericVector;
using Rcpp::log;
using Rcpp::pnorm;
using std::sqrt;
using std::log;

// [[Rcpp::export]]
NumericVector put_option_pricer_rcpp(NumericVector s, double k, double r, double y, double t, double sigma) {

    NumericVector d1 = (log(s / k) + (r - y + sigma * sigma / 2.0) * t) / (sigma * sqrt(t));
    NumericVector d2 = d1 - sigma * sqrt(t);
    
    NumericVector V = pnorm(-d2) * k * exp(-r * t) - s * exp(-y * t) * pnorm(-d1);
    return V;
}
{% endhighlight %}

We can call this from R as well:


{% highlight r %}
# Valuation with 1 stock price
put_option_pricer_rcpp(s = 55, 60, .01, .02, 1, .05)
{% endhighlight %}



<pre class="output">
[1] 5.52021
</pre>



{% highlight r %}
# Valuation across multiple prices
put_option_pricer_rcpp(s = 55:60, 60, .01, .02, 1, .05)
{% endhighlight %}



<pre class="output">
[1] 5.52021 4.58142 3.68485 2.85517 2.11883 1.49793
</pre>

Finally, let's look at
[RcppArmadillo](http://dirk.eddelbuettel.com/code/rcpp.armadillo.html). Armadillo has a
number of object types, including `mat`, `colvec`, and `rowvec`. Here, we just use
`colvec` to represent a column vector of prices. By default in Armadillo, `*` represents
matrix multiplication, and `%` is used for element wise multiplication. We need to make
this change to element wise multiplication in 1 place, but otherwise the changes are just
switching out the types and the sugar functions for Armadillo specific functions.

Note that the `arma::normcdf()` function is in the upcoming release of
[RcppArmadillo](http://dirk.eddelbuettel.com/code/rcpp.armadillo.html), which is
`0.8.400.0.0` at the time of writing and still in CRAN's incoming. It also requires the
`C++11` plugin.


{% highlight cpp %}
#include <RcppArmadillo.h>

// [[Rcpp::depends(RcppArmadillo)]]
// [[Rcpp::plugins(cpp11)]]

using arma::colvec;
using arma::log;
using arma::normcdf;
using std::sqrt;
using std::log;


// [[Rcpp::export]]
colvec put_option_pricer_arma(colvec s, double k, double r, double y, double t, double sigma) {
  
    colvec d1 = (log(s / k) + (r - y + sigma * sigma / 2.0) * t) / (sigma * sqrt(t));
    colvec d2 = d1 - sigma * sqrt(t);
    
    // Notice the use of % to represent element wise multiplication
    colvec V = normcdf(-d2) * k * exp(-r * t) - s * exp(-y * t) % normcdf(-d1); 

    return V;
}
{% endhighlight %}

Use from R:


{% highlight r %}
# Valuation with 1 stock price
put_option_pricer_arma(s = 55, 60, .01, .02, 1, .05)
{% endhighlight %}



<pre class="output">
        [,1]
[1,] 5.52021
</pre>



{% highlight r %}
# Valuation across multiple prices
put_option_pricer_arma(s = 55:60, 60, .01, .02, 1, .05)
{% endhighlight %}



<pre class="output">
        [,1]
[1,] 5.52021
[2,] 4.58142
[3,] 3.68485
[4,] 2.85517
[5,] 2.11883
[6,] 1.49793
</pre>

Finally, we can run a speed test to see which comes out on top.


{% highlight r %}
s <- matrix(seq(0, 100, by = .0001), ncol = 1)

rbenchmark::benchmark(R = put_option_pricer(s, 60, .01, .02, 1, .05),
                      Arma = put_option_pricer_arma(s, 60, .01, .02, 1, .05),
                      Rcpp = put_option_pricer_rcpp(s, 60, .01, .02, 1, .05), 
                      order = "relative", 
                      replications = 100)[,1:4]
{% endhighlight %}



<pre class="output">
  test replications elapsed relative
2 Arma          100   6.409    1.000
3 Rcpp          100   7.917    1.235
1    R          100   9.091    1.418
</pre>

Interestingly, [Armadillo](http://arma.sf.net) comes out on top here on this (multi-core)
machine (as Armadillo uses OpenMP where available in newer versions). But the difference
is slender, and there is certainly variation in repeated runs. And the nicest thing about
all of this is that it shows off the "embarassment of riches" that we have in the R and
C++ ecosystem for multiple ways of solving the same problem.
