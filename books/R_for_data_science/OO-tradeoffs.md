# Trade-offs {#oo-tradeoffs}



You now know about the three most important OOP toolkits available in R. Now that you understand their basic operation and the principles that underlie them, we can start to compare and constrast the systems in order to understand their strengths and weaknesses. This will help you pick the system that is most likely to solve new problems.

When picking an OO system, I recommend that you default to S3. S3 is simple, and widely used throughout base R and CRAN. While it's far from perfect, its idiosyncracries are well understood and there are known approaches to overcome most shortcomings. If you have an existing background in programming you are likely to lean towards R6 because it will feel familiar. I think you should resist this tendency for two reasons. Firstly, if you use R6 it's very easy to create an non-idiomatic API that will feel very odd to native R users, and will have surprising pain points because of the reference semantics. Secondly, if you stick to R6, you'll lose out on learning a new way of thinking about OOP that gives you a new set of tools for solving problems. 

This chapter is divided into two parts. [S4 vs S3](#s3-s4) compares S3 and S4. In brief, S4 is more formal and tends to require more upfront planning. That makes it more suitable for big projects developed by teams, not individuals. [R6 vs S3](#s3-r6) compares S3 and R6. This section is quite long because these two systems are fundamentally different and there are a number of tradeoffs that you need to consider.

## S4 vs S3 {#s3-s4}

Once you've mastered S3, S4 is relatively easy to pick up: the underlying ideas are the same, S4 is just more formal, more strict, and more verbose. The strictness and formality of S4 make it well suited for large teams. Since more structure is provided by the system itself, there is less need for convention, and new contributors don't need as much training. S4 tends to require more upfront design than S3, and this investment tends to be more likely to pay off on larger projects because greater resources are available.

One large team effort where S4 is used to good effect is Bioconductor. Bioconductor is similar to CRAN: it's a way of sharing packages amongst a wider audience. Bioconductor is smaller than CRAN (~1,300 vs ~10,000 packages, July 2017) and the packages tend to be more tightly integrated because of the shared domain and because Bioconductor has a stricter review process. Bioconductor packages are not required to use S4, but most will because the key data structures (e.g. SummarizedExperiment, IRanges, DNAStringSet) are built using S4.



S4 is also a good fit when you have a complicated system of interrelated objects, and it's possible to minimise code duplication through careful implementation of methods. The best example of this use of S4 is the Matrix package by Douglas Bates and Martin M??chler. It is designed to efficiently store and compute with many different types of sparse and dense matrices. As of version 1.2.12, it defines 102 classes, 21 generic functions, and 1993 methods. To give you some idea of the complexity, a small subset of the class graph is shown in Figure \@ref(fig:matrix-classes).

<div class="figure" style="text-align: center">
<img src="diagrams/s4-matrix-dsparseMatrix.png" alt="A small subset of the Matrix class graph showing the inheritance of sparse matrices. Each concrete class inherits from two virtual parents: one that describes how the data is stored (C = column oriented, R = row oriented, T = tagged) and one that describes any restriction on the matrix (s = symmetric, t = triangle, g = general)" width="80%" />
<p class="caption">(\#fig:matrix-classes)A small subset of the Matrix class graph showing the inheritance of sparse matrices. Each concrete class inherits from two virtual parents: one that describes how the data is stored (C = column oriented, R = row oriented, T = tagged) and one that describes any restriction on the matrix (s = symmetric, t = triangle, g = general)</p>
</div>

This domain is a good fit for S4 because there are often computational shortcuts for specific types of sparse matrix. S4 makes it easy to provide a general method that works for all inputs, and then provide a more specialised methods where the pair of data structures allow for a more efficient implementation. This requires careful planning to avoid method dispatch ambiguity, but the planning pays off for complicated systems.

The biggest challenge to using S4 is the combination of increased complexity and absence of a single source of documentation. S4 is a complex system and it can be challenging to use effectively in practice. This wouldn't be such a problem if S4 documentation wasn't scattered through R documentation, books, and websites. S4 needs a book length treatment, but that book does not (yet) exist. (The documentation for S3 is no better, but the lack is less painful because S3 is much simpler.)

## R6 vs S3  {#s3-r6}

R6 is a profoundly different OO system from S3 and S4 because it is built on encapsulated objects, rather than generic functions. Additionally R6 objects have reference semantics, which means that they can be modified in place. These two big differences have a number of non-obvious consequences which we'll explore in this chapter:

* A generic is a regular function so lives in the global namespace. A R6 method 
  belongs to an object so lives in a local namespace. This influences how we
  think about naming.

* R6's reference semantics allow methods to simultaneously return a value
  and update the object. This solves a painful problem called "threading state".
  
* You invoke an R6 method using `$`, which is an infix operator. If you set up
  your methods correctly you can use chains of method calls as an alternative
  to the pipe.

(All these trade-offs apply in general to immutable functional OOP vs mutable encapsulated OOP so also serve as a discussion of the tradeoffs between S3 and reference classes, and S3 and OOP in languages like Python.)

### Namespacing

One non-obvious difference between S3 and R6 is the "space" in which methods are found:

* Generic functions are global: all packages share the same namespace. 
* Encapsulated methods are local: methods are bound to a single object.

The advantage of a global namespace is that multiple packages can use exactly the same verbs for working with different types of objects. Generic functions provide a uniform API that makes it easier to perform typical actions with a new object because there are strong naming conventions. This works well for data analysis because you often want to do the same thing to different types of objects. In particular, this is one reason that R's modelling system is so useful: regardless of where the model has been implemented you always work with it using the same set of tools (`summary()`, `predict()`, ...).

The disadvantage of a global namespace is that forces you to think more deeply about naming. You want to avoid multiple generics with the same name in different pakages because it requires the user to type `::` frequently. This can be hard because function names are usually English verbs, and verbs often have multiple meanings. Take `plot()` for example:


```r
plot(data)       # plot some data
plot(bank_heist) # plot a crime
plot(land)       # create a new plot of land
plot(movie)      # extract plot of a movie
```

Generally, you should avoid defining methods like this. Don't use homonyms of the original generic, but instead define a new generic. This problem doesn't occur with R6 methods because they are scoped to the object. The following code is fine, because there is no implication that the plot method of two different R6 objects has the same meaning:


```r
data$plot()
bank_heist$plot()
land$plot()
movie$plot()
```

These considerations also apply to the arguments to the generic. S3 generics must have the same core arguments, which mean they generally have to have non-specific names like `x` or `.data`. S3 generics generally need `...` to pass on additional arguments to methods, but this has the downside that mispelled argument names will not create an error. In comparison, R6 methods can vary more widely and use more specific and evocative argument names.

A secondary advantage of local namespacing is that creating an R6 method is very cheap. Most encapsulated OO languages encourage you to create many small methods, each doing one thing well with an evocative name. Creating a new S3 method is more expensive, because you may also have to create a generic, and think about the naming issues described above. That means that the advice to create many small methods does not apply to S3. It's still a good idea to break your code down into small, easily understood chunks, but they should generally just be regular functions, not methods.

### Threading state

One challenge of programming with S3 is when you want to both return a value and modify the object. This violates our guideline that a function should either be called for its return value or for its side effects, but is necessary in a handful of cases. For example, imagine you want to create a __stack__ of objects. A stack has two main methods: 

* `push()` adds a new object to the top of the stack.
* `pop()` returns the top most value, and removes it from the stack. 

The implementation of the constructor and the `push()` method is straightforward. A stack contains a list of items, and pushing an object to the stack simply appends to this list.


```r
new_stack <- function(items = list()) {
  structure(list(items = items), class = "stack")
}

push <- function(x, y) {
  x$items <- c(x$items, list(y))
  x
}
```

(Note that I haven't created a real method for `push()` because making it generic would just make this example more complicated for no real benefit.)

Implementing `pop()` is more challenging because it has to both return a value (the object at the top of the stack), and have a side-effect (remove that object from that top). Since we can't modify the input object in S3 we need to return two things: the value, and the updated object.


```r
pop <- function(x) {
  n <- length(x$items)
  
  item <- x$items[[n]]
  x$items <- x$items[-n]
  
  list(item = item, x = x)
}
```

This leads to rather awkward usage:


```r
s <- new_stack()
s <- push(s, 10)
s <- push(s, 20)

out <- pop(s)
out$item
#> [1] 20
s <- out$x
s
#> $items
#> $items[[1]]
#> [1] 10
#> 
#> 
#> attr(,"class")
#> [1] "stack"
```

This problem is known as __threading state__ or __accumulator programming__,  because no matter how deeply the `pop()` is called, you have to feed the modified stack object all the way back to where the stack lives.

One way that other FP languages deal with this challenge is to provide a "multiple assign" (or destructing bind) operator that allows you to assign multiple values in a single step. The zeallot R package, by Nathan and Paul Teetor, provides multi-assign for R with `%<-%`. This makes the code more elegant, but doesn't solve the key problem:


```r
library(zeallot)

c(value, s) %<-% pop(s)
value
#> [1] 10
```

An R6 implementation of a stack is simpler because `$pop()` can modify the object in place, and return only the top-most value:


```r
Stack <- R6::R6Class("Stack", list(
  items = list(),
  push = function(x) {
    self$items <- c(self$items, x)
    invisible(self)
  },
  pop = function() {
    item <- self$items[[self$length()]]
    self$items <- self$items[-self$length()]
    item
  },
  length = function() {
    length(self$items)
  }
))
```

This leads to more natural code:


```r
s <- Stack$new()
s$push(10)
s$push(20)
s$pop()
#> [1] 20
```

### Method chaining {#tradeoffs-pipe}

The pipe, `%>%`, is useful because it provides an infix operator that makes it easy to compose functions from left-to-right. Interestingly, the pipe is not so important for R6 objects because they already use an infix operator: `$`. This allows the user to chain together multiple method calls in a single expression, a technique known as __method chaining__:


```r
s <- Stack$new()
s$
  push(10)$
  push(20)$
  pop()
#> [1] 20
```

This technique is commonly used in other programming languages, like Python and Javascript, and is made possible with one convention: any R6 method that is primarily called for its side-effects (usually modifying the object) should return `invisible(self)`. 

The primary advantage of method chaining is that you can get useful autocomplete; the primary disadvantage is that only the creator of the class can add new methods (and there's no way to use multiple dispatch).
