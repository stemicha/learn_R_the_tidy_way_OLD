# Base types {#base-types}



To talk about objects and OOP in R we need to first deal with a fundamental confusion: we use the word object to mean two different things. In this book so far, we've used object in a general sense, as captured by John Chambers' pithy quote: "Everything that exists in R is an object". However, while everything _is_ an object, not everything is "object-oriented". This confusion arises because the base objects come from S, and were developed before anyone was thinking that S might need an OOP system. The tools and nomenclature evolved organically over many years without a single guiding principle.

Most of the time, the distinction between objects and object-oriented objects is not important. But here we need to get into the nitty gritty details so we'll use the terms __base objects__ and __OO objects__ to distinguish them.

<img src="diagrams/oo-venn.png" width="305" style="display: block; margin: auto;" />

We'll also discuss the `is.*` functions here. These functions are used for many purposes, but are commonly used to determine if an object has a specific base type.

## Base objects vs OO objects

To tell the difference between a base and an OO object, use `is.object()`:


```r
# A base object:
is.object(1:10)
#> [1] FALSE

# An OO object
is.object(mtcars)
#> [1] TRUE
```

(This function would be better called `is.oo()` because it tells you if an object is a base object or a OO object.)

The primary attribute that distinguishes between base and OO object is the "class". Base objects do not have a class attribute:


```r
attr(1:10, "class")
#> NULL

attr(mtcars, "class")
#> [1] "data.frame"
```

Note that `attr(x, "class")` and `class(x)` do not always return the same thing, as `class()` returns a value, not `NULL`, for base objects. We'll talk about exactly what it does return in the next chapter.

## Base types {#base-types-2}

While only OO objects have a class attribute, every object has a __base type__:


```r
typeof(1:10)
#> [1] "integer"

typeof(mtcars)
#> [1] "list"
```

Base types do not form an OOP system because functions that behave differently for different base types are primarily written in C, where dispatch occurs using switch statements. This means only R-core can create new types, and creating a new type is a lot of work. As a consequence, new base types are rarely added. The most recent change, in 2011, added two exotic types that you never see in R, but are needed for diagnosing memory problems (`NEWSXP` and `FREESXP`). Prior to that, the last type added was a special base type for S4 objects (`S4SXP`) added in 2005.

<!-- https://github.com/wch/r-source/blob/bf0a0a9d12f2ce5d66673dc32cd253524f3270bf/src/include/Rinternals.h#L149-L180 -->

In total, there are 25 different base types. They are listed below, loosely grouped according to where they're discussed in this book.

*   The vectors: NULL, logical, integer, double, complex, character,
    list, raw.
    
    
    ```r
    typeof(1:10)
    #> [1] "integer"
    typeof(NULL)
    #> [1] "NULL"
    typeof(1i)
    #> [1] "complex"
    ```

*   Functions: closure (regular R functions), special (internal functions), 
    builtin (primitive functions) and environment.
    
    
    ```r
    typeof(mean)
    #> [1] "closure"
    typeof(`[`)
    #> [1] "special"
    typeof(sum)    
    #> [1] "builtin"
    typeof(globalenv())
    #> [1] "environment"
    ```
    
*   Language components: symbol (aka names), language (usually called calls),
    pairlist (used for function arguments).

    
    ```r
    typeof(quote(a))
    #> [1] "symbol"
    typeof(quote(a + 1))
    #> [1] "language"
    typeof(formals(mean))
    #> [1] "pairlist"
    ```
 
    "Expression" is a special purpose type that's only returned by  `parse()` 
    and `expression()`. They are not needed in user code.
        
*  There are a few esoteric types that are important for C code but not 
   generally available at the R level: externalptr, weakref, bytecode, S4,
   promise, "...", and any.

You may have heard of `mode()` and `storage.mode()`. I recommend ignoring these functions because they just provide S compatible aliases of `typeof()`. Read the source code if you want to understand exactly what they do. \indexc{mode()}

## The is functions

<!-- https://github.com/wch/r-source/blob/880337b753960bf77c6ccd8badca634e0f2a4914/src/main/coerce.c#L1764 -->

This is also a good place to discuss the `is` functions because they're often used to check if an object has a specific type:


```r
is.function(mean)
#> [1] TRUE
is.primitive(sum)
#> [1] TRUE
```

"Is" functions are often surprising because there are several different classes, they often have a few special cases, and their names are historical so don't always reflect the usage in this book. They fall roughly into six classes:

*   A specific value of `typeof()`:
    `is.call()`, `is.character()`, `is.complex()`, 
    `is.double()`, `is.environment()`, `is.expression()`,
    `is.list()`, `is.logical()`, `is.name()`, `is.null()`, `is.pairlist()`,
    `is.raw()`, `is.symbol()`.
    
    `is.integer()` is almost in this class, but it specifically checks for the
    absense of a class attribute containing "factor". Also note that 
    `is.vector()` belongs to the "attributes" class, and `is.numeric()` is 
    described specially below.
  
*   A set of possible of base types: 

    * `is.atomic()` = logical, integer, double, characer, raw, and 
      (surprisingly) NULL.
      
    * `is.function()` = special, builtin, closure.
    
    * `is.primitive()` = special, builtin.
    
    * `is.language()` = symbol, language, expression.
    
    * `is.recursive()` = list, language, expression.

*   Attributes: 

    * `is.vector(x)` tests that `x` has no attributes apart from names.
      It does __not__ check if an object is an atomic vector or list.
      
    * `is.matrix(x)` tests if `length(dim(x))` is 2.
    
    * `is.array(x)` tests if `length(dim(x))` is 1 or 3+.
    
*   Has an S3 class: `is.data.frame()`, `is.factor()`, `is.numeric_version()`,
    `is.ordered()`, `is.package_version()`, `is.qr()`, `is.table()`.

*   Vectorised mathematical operation: 
    `is.finite()`, `is.infinite()`, `is.na()`, `is.nan()`.

*   Finally there are a bunch of special purpose functions that don't 
    fall into any other category: 
        
    * `is.loaded()`: tests if a C/Fortran subroutine is loaded.
    * `is.object()`: discussed above.
    * `is.R()` and `is.single()`: are included for S+ compatibility
    * `is.unsorted()` tests if a vector is unsorted.
    * `is.element(x, y)` checks if `x` is an element of `y`: it's even more 
       different as it takes two arguments, unlike every other `is`. function.

One function, `is.numeric()`, is sufficiently complicated and important that it needs a little extra discussion. The complexity comes about because R uses "numeric" to mean three slightly different things:

1.  In some places it's used as an alias for "double".  For example
    `as.numeric()` is identical to `as.double()`, and `numeric()` is
    identical to `double()`.
    
    R also occasionally uses "real" instead of double; `NA_real_` is the one 
    place that you're likely to encounter this in practice.
    
1.  In S3 and S4 it is used to mean either integer or double. We'll
    talk about `s3_class()` in the next chapter:

    
    ```r
    sloop::s3_class(1)
    #> [1] "double"  "numeric"
    sloop::s3_class(1L)
    #> [1] "integer" "numeric"
    ```

1.  In `is.numeric()` it means an object built on a base type of integer or 
    double that is not a factor (i.e. it is a number and behaves like a number).
    
    
    ```r
    is.numeric(1)
    #> [1] TRUE
    is.numeric(1L)
    #> [1] TRUE
    is.numeric(factor("x"))
    #> [1] FALSE
    ```

