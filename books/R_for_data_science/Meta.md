# (PART) Metaprogramming {-}

# Introduction {#meta}



> "Flexibility in syntax, if it does not lead to ambiguity, would seem a
> reasonable thing to ask of an interactive programming language."
>
> --- Kent Pitman

Compared to most programming languages, one of the most surprising things about R is its capability for metaprogramming: the ability of code to inspect and modify other code. Metaprogramming is particularly important in R because R is not just a programming language; it's also an environment for doing interactive data analysis. Metaprogramming is useful for interactive exploration because it allows packages to create evaluation environments that use slightly different rules to usual code. This allows packages like ggplot2 and dplyr to create embedded domain specific languages tailored for solving specific data analysis problems.

Embedded DSLs take advantage of a host language's parsing and execution framework, but adjust the semantics to make them more suitable for a specific task. DSLs are a very large topic, and this chapter will only scratch the surface, focussing on important implementation techniques rather than on how you might come up with the language in the first place. If you're interested in learning more, I highly recommend [_Domain Specific Languages_](http://amzn.com/0321712943?tag=devtools-20) by Martin Fowler. It discusses many options for creating a DSL and provides many examples of different languages. \index{domain specific languages}

A common use of metaprogramming is to allow you to use names of variables in a dataframe as if they were objects in the environment. This makes interactive exploration more fluid at the cost of introducing some minor ambiguity. For example, take `base::subset()`. It allows you to pick rows from a dataframe based on the values of their observations:


```r
data("diamonds", package = "ggplot2")
subset(diamonds, x == 0 & y == 0 & z == 0)
#>       carat       cut color clarity depth table price x y z
#> 11964  1.00 Very Good     H     VS2  63.3    53  5139 0 0 0
#> 15952  1.14      Fair     G     VS1  57.5    67  6381 0 0 0
#> 24521  1.56     Ideal     G     VS2  62.2    54 12800 0 0 0
#> 26244  1.20   Premium     D    VVS1  62.1    59 15686 0 0 0
#> 27430  2.25   Premium     H     SI2  62.8    59 18034 0 0 0
#> 49557  0.71      Good     F     SI2  64.1    60  2130 0 0 0
#> 49558  0.71      Good     F     SI2  64.1    60  2130 0 0 0
```

(Base R functions like `subset()` and `transform()` inspired the development of dplyr.)

`subset()` is considerably shorter than the equivalent code using `[` and `$` because you only need to provide the name of the data frame once:


```r
diamonds[diamonds$x == 0 & diamonds$y == 0 & diamonds$z == 0, ]
#>       carat       cut color clarity depth table price x y z
#> 11964  1.00 Very Good     H     VS2  63.3    53  5139 0 0 0
#> 15952  1.14      Fair     G     VS1  57.5    67  6381 0 0 0
#> 24521  1.56     Ideal     G     VS2  62.2    54 12800 0 0 0
#> 26244  1.20   Premium     D    VVS1  62.1    59 15686 0 0 0
#> 27430  2.25   Premium     H     SI2  62.8    59 18034 0 0 0
#> 49557  0.71      Good     F     SI2  64.1    60  2130 0 0 0
#> 49558  0.71      Good     F     SI2  64.1    60  2130 0 0 0
```

Functions like `subset()` are often said to use __non-standard evalution__, or NSE for short. \index{non-standard evaluation} That's because they evaluate one (or more) of their arguments in a non-standard way. For example, if you take the second argument to `subset()` above and try and evaluate it directly, the code will not work:


```r
x == 0 | y == 0 | z == 0
```

As you might guess, defining these tools by what they are not (standard evaluation) is somewhat problematic. Additionally, implementation of the underlying ideas has occurred piecemeal over the last twenty years. These two forces tend to make base R code for NSE harder to understand than it could be; the key ideas are obscured by unimportant details. To avoid these problems here, these chapters will focus on functions from the __rlang__ package. Then once you have the basic ideas, I'll also show you the equivalent base R code so you can more easily understand existing code.

In this section of the book, you'll learn about the three big ideas that underpin NSE:

* In __Expressions__, [Expressions], shows the hierarchical structure of R code.
  You'll learn how to visualise the hierarchy for arbitrary code, how the rules 
  of R's grammar convert linear sequences of characters into a tree, and how to 
  use recursive functions to work with code trees.
  
* In __Quotation__, [Quotation], you'll learn to use tools from rlang to capture 
  unevaluated function arguments. You'll also learn about quasiquotation, which 
  provides a set of techniques for "unquoting" input, and see how you can 
  generate R code by calling functions instead of writing it by hand.
  
* In __Evaluation__, [Evaluation], you'll learn about the inverse of quotation:
  evaluation. Here you'll learn about an important data structure, the quosure,
  which ensures correct evaluation by capturing both the code to evaluate, and
  the environment in which to evaluate it. This chapter will show you how put 
  all the pieces together to understand how NSE works, and how to write your 
  own functions that work like `subset()`.

This part of the book concludes with a case study in [DSLs](#dsls). This chapter pulls together the threads of metaprogramming and shows how you can use R to create two __domain specific languages__ for translating R code into HTML and LATEX. You'll learn important techniques similar to those used by shiny and dplyr.

Each chapter follows the same basic structure. You'll get the lay of the land in introduction, then see a motivating example. Next you'll learn the big ideas using functions from rlang, and then we'll circle back to talk about how those ideas are expressed in base R. Each chapter finishes with a case study, using the ideas to solve a bigger problem.

If you're reading these chapters primarily to better understand tidy evaluation so you can better program with the tidyverse, I'd recommend just reading the first 2-3 sections of each chapter; skip the sections about base R and more advanced techniques. 
