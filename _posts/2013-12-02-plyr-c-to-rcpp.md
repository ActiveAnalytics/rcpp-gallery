---
title: Converting C code to C++ code&#58; An example from plyr
author: Hadley Wickham and Dirk Eddelbuettel
license: GPL (>= 2)
tags: stl basics
summary: C++ allows for more idiomatic code.
layout: post
src: 2013-12-02-plyr-c-to-rcpp.Rmd
---

The [plyr](http://cran.r-project.org/package=plyr) package uses a couple of 
small C functions to optimise a number of particularly bad bottlenecks. 
Recently, two functions were converted to C++. This was mostly stimulated by a 
segmentation fault caused by some large inputs to the `split_indices()` 
function: rather than figuring out exactly what was going wrong with the 
complicated C code, it was easier to rewrite with simple, correct C++ code.

The job of `split_indices()` is simple: given a vector of integers, `x`,
it returns a list where the `i`-th element of the list is an integer vector
containing the positions of `x` equal to `i`. This is a useful building block
for many of the functions in [plyr](http://cran.r-project.org/package=plyr).

It is fairly easy to see what is going on the in the C++ code:


{% highlight cpp %}
#include <Rcpp.h>

using namespace Rcpp;

// [[Rcpp::exports]]
std::vector<std::vector<int> > split_indices(IntegerVector x, int n = 0) {
    if (n < 0) stop("n must be a positive integer");
  
    std::vector<std::vector<int> > ids(n);
  
    int nx = x.size();
    for (int i = 0; i < nx; ++i) {
        if (x[i] > n) {
           ids.resize(x[i]);
        }
    
        ids[x[i] - 1].push_back(i + 1);
    }
  
    return ids;
}
{% endhighlight %}


* We create a `std::vector` of integers called `out`. This will grow
efficiently as we add new values, and Rcpp will automatically convert to a
list of integer vectors when returned to R. 

* The loop iterates through each element of `x`, adding its index to the end
of `out`. It also makes sure that `out` is long enough. (The plus and minus
ones are needed because C++ uses 0 based indices and R uses 1 based
indices.)

The code is simple, easy to understand (if one is a little familiar with
the STL), and performant.  Compare it to the original C code:


{% highlight cpp %}
#include <R.h>
#include <Rdefines.h>

SEXP split_indices(SEXP group, SEXP n) {
    SEXP vec;
    int i, j, k;

    int nlevs = INTEGER(n)[0];
    int nobs = LENGTH(group);  
    int *pgroup = INTEGER(group);
  
    // Count number of cases in each group
    int counts[nlevs];
    for (i = 0; i < nlevs; i++)
        counts[i] = 0;
    for (i = 0; i < nobs; i++) {
        j = pgroup[i];
        if (j > nlevs) error("n smaller than largest index");
        counts[j - 1]++;
    }

    // Allocate storage for results
    PROTECT(vec = allocVector(VECSXP, nlevs));
    for (i = 0; i < nlevs; i++) {
        SET_VECTOR_ELT(vec, i, allocVector(INTSXP, counts[i]));
    }

    // Put indices in groups
    for (i = 0; i < nlevs; i++) {
        counts[i] = 0;
    }
    for (i = 0; i < nobs; i++) {
        j = pgroup[i] - 1;
        k = counts[j];
        INTEGER(VECTOR_ELT(vec, j))[k] = i + 1;
        counts[j]++;
    }
    UNPROTECT(1);
    return vec;
}
{% endhighlight %}


This function is almost three times as long, and has a bug in it.  It is
substantially more complicated because it:

* has to take care of memory management with `PROTECT` and `UNPROTECT`;
  Rcpp takes care of this for us

* needs an additional loop through the data to determine how long each
  vector should be; the `std::vector` grows efficiently and eliminates this
  problem

Conversion to C++ can make code shorter and easier to understand and maintain,
while remaining just as performant.
