# Expressions



To compute on the language, we first need to understand its structure. That requires some new vocabulary, some new tools, and some new ways of thinking about R code. \index{expressions} The first thing you'll need to understand is the distinction between an operation and its result. Take this code, which takes a variable `x` multiplies it by 10 and saves the result to a new variable called `y`. It doesn't work because we haven't defined a variable called `x`:


```r
y <- x * 10
#> Error in eval(expr, envir, enclos): Objekt 'x' nicht gefunden
```

It would be nice if we could capture the intent of the code, without executing the code. In other words, how can we separate our description of the action from performing it? One way is to use `rlang::expr()`: \indexc{expr()}


```r
z <- expr(y <- x * 10)
z
#> y <- x * 10
```

`expr()` returns a quoted __expression__: the R code that captures our intent.

In this chapter, you'll learn about the structure of those expressions, which will also help you understand how R executes code. Later, we'll learn about `eval()` which allows you to take such an expression and perform, or __evaluate__, it:


```r
x <- 4
eval(z)
y
#> [1] 40
```

## Abstract syntax trees

Quoted expressions are also called abstract syntax trees (AST) because the structure of code is fundamentally hierarchical and can be naturally represented as a tree. To make that more obvious we're going to introduce some graphical conventions, illustrated with the very simple call `f(x, "y", 1)`. \index{abstract syntax tree}

<img src="diagrams/expression-simple.png" width="201" style="display: block; margin: auto;" />
  
*   Function __calls__ define the hierarchy of the tree. Calls are shown
    with an orange square. The first child (`f`) is the function that gets 
    called. The second and subsequent children (`x`, `"y"`, and `1`) are the 
    arguments. 
  
    __NB__: Unlike many tree diagrams the order of the children is important: 
    `f(x, 1)` is not the same as `f(1, x)`.
    
*   The leaves of the tree are either __symbols__, like `f` and `x`, or 
    __constants__ like `1` or `"y"`. Symbols have a purple border and rounded 
    corners. Constants, which are atomic vectors of length one, have black 
    borders and square corners. Strings are always surrounded in quotes to
    emphasise their difference to symbols --- more on that later.

Drawing these diagrams by hand takes me some time, and obviously you can't rely on me to draw diagrams for your own code. I'll supplement the hand-drawn trees with trees drawn by `lobstr::ast()`. `ast()` tries to make trees as similar as possible to my hand-drawn trees, while respecting the limitations of the console. Let's use `ast()` to display the tree above:


```r
lobstr::ast(f(x, "y", 1))
#> ??????f 
#> ??????x 
#> ??????"y" 
#> ??????1
```

Calls get an orange square, symbols are bold and purple, and strings are surrounded by quote marks. (The formatting is not currently shown in the rendered book, but you can see it if you run the code yourself.)

`ast()` supports "unquoting" with `!!` (pronounced bang-bang). We'll talk about unquoting in detail in the next chapter; for now note that it's useful if you've already used `expr()` to capture the expression.


```r
x <- expr(f(x, "y", 1))

# not useful!
lobstr::ast(x)
#> x

# what we want
lobstr::ast(!!x)
#> ??????f 
#> ??????x 
#> ??????"y" 
#> ??????1
```

For more complex code, you can also use RStudio's tree viewer to explore the AST interactively, e.g. `View(expr(y <- x * 10))`.

### Infix vs. prefix calls

Every call in R can be written in tree form, even if it doesn't look like it at first glance. Take `y <- x * 10` again: what are the functions that are being called? It not as easy to spot as `f(x, 1)` because this expression contains two calls in __infix__ form: `<-` and `*`. Infix functions come **in**between their arguments (so an infix function can only have two arguments), whereas most functions in R are __prefix__ functions where the name of the function comes first [^1]. 

[^1]: Some programming languages use __postfix__ calls where the name of the function comes last. If you ever used an old HP calculator, you might have fallen in love with reverse Polish notation, postfix notation for algebra. There is also a family of "stack"-based programming languages descending from Forth which takes this idea as far as it might possibly go.

In R, any infix call can be converted to a prefix call if you escape the function name with backticks. That means that these two lines of code are equivalent:


```r
y <- x * 10
`<-`(y, `*`(x, 10))
```

And they have this AST:

<img src="diagrams/expression-prefix.png" width="216" style="display: block; margin: auto;" />


```r
lobstr::ast(y <- x * 10)
#> ??????`<-` 
#> ??????y 
#> ????????????`*` 
#>   ??????x 
#>   ??????10
```

You might remember that code like `names(x) <- y` ends up calling the `names<-` function. That is not reflected in the parse tree because the translation needs to happen later, due to the complexities of nested assignments like `names(x)[2] <- "z"`.


```r
lobstr::ast(names(x) <- y)
#> ??????`<-` 
#> ????????????names 
#> ??? ??????x 
#> ??????y
```

### Special forms

R has a small number of other syntactical constructs that don't look like either prefix or infix function calls. These are called __special forms__ and include `function`, the control flow operators (`if`, `for`, `while`, `repeat`), and parentheses (`{`, `(`, `[[`, and `[`). These can also be written in prefix form, and hence appear in the same way in the AST:


```r
lobstr::ast(function(x, y) {
  if (x > y) {
    x
  } else {
    y
  }
})
#> ??????`function` 
#> ????????????x = `` 
#> ??? ??????y = `` 
#> ????????????`{` 
#> ??? ????????????`if` 
#> ???   ????????????`>` 
#> ???   ??? ??????x 
#> ???   ??? ??????y 
#> ???   ????????????`{` 
#> ???   ??? ??????x 
#> ???   ????????????`{` 
#> ???     ??????y 
#> ??????<inline srcref>
```

### Function factories

Another small detail we need to consider are calls like `f()()`. The first component of the call is usually a symbol:


```r
lobstr::ast(f(a, 1))
#> ??????f 
#> ??????a 
#> ??????1
```

But if you are using a function factory, a function that returns another function, the first component might be another call:


```r
lobstr::ast(f()(a, 1))
#> ????????????f 
#> ??????a 
#> ??????1
```

(See [Function factories] for more details)

Of course that function might also take arguments:


```r
lobstr::ast(f(b, 2)())
#> ????????????f 
#>   ??????b 
#>   ??????2
```

These forms are relatively rare, but it's good to be able to recognise them when they crop up. 

### Argument names

So far the examples have only used unnamed arguments. Named arguments don't change the parsing rules, but just add some additional metadata:


```r
lobstr::ast(mean(x = mtcars$cyl, na.rm = TRUE))
#> ??????mean 
#> ??????x = ??????`$` 
#> ???     ??????mtcars 
#> ???     ??????cyl 
#> ??????na.rm = TRUE
```

(And note the appearance of another infix function: `$`)

### Exercises

1.  Use `ast()` and experimentation to figure out the three arguments to an
    `if()` call. What would you call them? Which components are required? 
    
1.  What are the arguments to the `for()` and `while()` calls? 

1.  What does the call tree of an `if` statement with multiple `else if` 
    conditions look like? Why?
    
1.  Two arithmetic operators can be used in both prefix and infix style.
    What are they?

## R's grammar

The process by which a computer language takes a sequence of tokens (like `x`, `+`, `y`) and constructs a tree is called __parsing__, and it is governed by a set of rules known as a __grammar__. In this section, we'll use `lobstr::ast()` to explore some of the details of R's grammar. 

If this is your first reading of the metaprogramming chapters, now is a good time to read the first sections of the next two chapters in order to get the big picture. Come back and learn more of the details once you've seen how all the big pieces fit together.

### Operator precedence and associativity

Infix functions introduce ambiguity in a way that prefix functions do not[^2]. The parser has to resolve two sources of ambiguity when parsing infix operators. First, what does `1 + 2 * 3` yield? Do you get 9 (i.e. `(1 + 2) * 3`), or 7 (i.e. `1 + (2 * 3)`).  Which of the two possible parse trees below does R use?

[^2]: These two sources of ambiguity do not exist without infix operators, which can be considered an advantage of purely prefix and postfix languages. It's interesting to compare a simple arithmetic operation in Lisp (prefix) and Forth (postfix). In Lisp you'd write `(+ (+ 1 2) 3))`; this avoids ambiguity by requiring parentheses everywhere. In Forth, you'd write `1 2 + 3 +`; this doesn't require any parentheses, at the cost of requiring more thought when reading.

<img src="diagrams/expression-ambig-order.png" width="408" style="display: block; margin: auto;" />

Programming langauges use conventions called __operator precedence__ to resolve this ambiguity. We can use `ast()` to see what R does: 


```r
lobstr::ast(1 + 2 * 3)
#> ??????`+` 
#> ??????1 
#> ????????????`*` 
#>   ??????2 
#>   ??????3
```

Predicting the precedence of arithmetic operations is usually easy because it's drilled into you in school and is consistent across the vast majority of programming languages. Predicting the precedence of other operators is harder. There's one particularly surprising finding in R: `!` has a much lower precedence (i.e. it binds less tightly) than you might expect. This allows you to write useful operations like:


```r
lobstr::ast(!x %in% y)
#> ??????`!` 
#> ????????????`%in%` 
#>   ??????x 
#>   ??????y
```

Another source of ambiguity is repeated usage of the same infix function. For example, is `1 + 2 + 3` equivalent to `(1 + 2) + 3` or to `1 + (2 + 3)`?  This normally doesn't matter because `x + y == y + x`,  i.e. addition is associative. However, some S3 classes define `+` in a non-associative way. For example, ggplot2 overloads `+` to build up a complex plot from simple pieces; this usage is non-associative because earlier layers are drawn underneath later layers.

In R, most operators are __left-associative__, i.e. the operations on the left are evaluated first:


```r
lobstr::ast(1 + 2 + 3)
#> ??????`+` 
#> ????????????`+` 
#> ??? ??????1 
#> ??? ??????2 
#> ??????3
```

R has over 30 infix operators divided into 18 precedence groups. While the details are descrbed in `?Syntax`, very few people have memorised the complete ordering. Indeed, if there's any confusion, use parentheses! 

These also appear in the AST, like all other special forms:


```r
lobstr::ast(1 + (2 + 3))
#> ??????`+` 
#> ??????1 
#> ????????????`(` 
#>   ????????????`+` 
#>     ??????2 
#>     ??????3
```

### Whitespace

R, in general, is not sensitive to white space. Most white space is not signficiant and is not recorded in the AST. `x+y` yields exactly the same AST as `x +        y`. This means that you're generally free to add whitespace to enhance the readability of your code. There's one major exception:


```r
lobstr::ast(y <- x)
#> ??????`<-` 
#> ??????y 
#> ??????x
lobstr::ast(y < -x)
#> ??????`<` 
#> ??????y 
#> ????????????`-` 
#>   ??????x
```

### Parsing a string

Most of the time you type code into the console, and R takes care of turning the characters you've typed into an AST. But occasionally you have code stored in a string, and you want to parse it yourself. You can do so using `rlang::parse_expr()`:


```r
x1 <- "y <- x + 10"
lobstr::ast(!!x1)
#> "y <- x + 10"

x2 <- rlang::parse_expr(x1)
x2
#> y <- x + 10
lobstr::ast(!!x2)
#> ??????`<-` 
#> ??????y 
#> ????????????`+` 
#>   ??????x 
#>   ??????10
```

(If you find yourself working with strings containing code very frequently, you should reconsider your work. Read the next chapter and consider if you can more safely generate expressions using quasiquotation.)

The base equivalent to `parse_expr()` is `parse()`. It is a little harder to use because it's specialised for parsing R code stored in files. That means you need supply your string to the `text` argument, and you'll get back an expression object (more on that shortly) that you'll need to subset:


```r
parse(text = x1)[[1]]
#> y <- x + 10
```

### Deparsing

The opposite of parsing is __deparsing__: you have an AST and you want a string that would generate it:


```r
z <- expr(y <- x + 10)
expr_text(z)
#> [1] "y <- x + 10"
```

Parsing and deparsing are not perfectly symmetrical because parsing throws away all information not directly related to the AST. This includes backticks around ordinary names, comments, and whitespace:


```r
cat(expr_text(expr({
  # This is a comment
  x <-             `x` + 1
})))
#> {
#>     x <- x + 1
#> }
```

Deparsing is often used to provide default names for data structures (like data frames), and default labels for messages or other output. rlang provides two helpers for those situations:


```r
z <- expr(f(x, y, z))

expr_name(z)
#> [1] "f(x, y, z)"
expr_label(z)
#> [1] "`f(x, y, z)`"
```

Be careful when using the base R equivalent, `deparse()`: it returns a character vector, and with one element for each line. Whenever you use it, remember that the length of the output might be greater than one, and plan accordingly.

### Exercises

1.  R uses parentheses in two slightly different ways as illustrated by 
    this simple call: `f((1))`. Compare and contrast the two uses. 
    
1.  `=` can also be used in two ways. Construct a simple example that shows 
    both uses.

1.  What does `!1 + !1` return? Why?

1.  Which arithmetic operation is right associative?

1.  Why does `x1 <- x2 <- x3 <- 0` work? There are two reasons.

1.  Compare `x + y %+% z` to `x ^ y %+% z`. What does that tell you about 
    the precedence of custom infix functions?

1.  `deparse()` produces vectors when the input is long. For example, the 
    following call produces a vector of length two:

    
    ```r
    expr <- expr(g(a + b + c + d + e + f + g + h + i + j + k + l + m +
      n + o + p + q + r + s + t + u + v + w + x + y + z))
    
    deparse(expr)
    ```

    What do `expr_text()`, `expr_name()`, and `expr_label()` do?

## Data structures

Now that you have a good feel for ASTs and how R's grammar helps to define them, it's time to look into more detail about the underlying implementation. In this section you'll learn about the data structures that R uses to implement the AST:

* Constants and symbols, the leaves of the tree.
* Calls, the branches of the tree.
* Pairlists, a historical data structure now only used for function arguments.

Before we continue, a word of caution about the naming conventions used in this book. Unfortunately, base R does not have a consistent set of conventions, so we've had to make our own. We use them consistently in the book and in rlang, but you'll need to remember some translations when reading base R documentation.

We use expression to refer collectively to the data structures that can appear in an AST (constant, symbol, call, pairlist). In base R, "expression" is a special type that is basically equivalent to a list of what we call expressions. To avoid confusion (as much as possible) we'll call these __expression objects__, and we'll discuss in [expression objects]. 

Base R does not have an equivalent term for our "expression". The closest is "language object", which includes symbols and calls, but not constants or pairlists. But note that `typeof()` and `str()` use "language" to mean call. Base R uses symbol and name interchangeably; we prefer symbol because "name" has other common meanings (e.g. the name of a variable).

### Constants and symbols

Constants and symbols are the leaves of the AST. Symbols represent variable names, and constants represent literal values.

Constants are "self-quoting" in the sense that the expression used to represent a constant is the constant itself:


```r
identical(expr("x"), "x")
#> [1] TRUE
identical(expr(TRUE), TRUE)
#> [1] TRUE
identical(expr(1), 1)
#> [1] TRUE
identical(expr(2), 2)
#> [1] TRUE
```

Symbols are used to represent variables. You can convert back and forth between symbols and the strings that represent them with `sym()` and `as_string()`:


```r
"x"
#> [1] "x"
sym("x")
#> x
as_string(sym("x"))
#> [1] "x"
```

#### The missing symbol

There's one special symbol that needs a little extra discussion: the empty symbol which is used to represent missing arguments. You can make it with  `missing_arg()` (or `expr()`):


```r
missing_arg()
```

The missing argument throws an error if the symbol it is bound to is accessed:


```r
m1 <- missing_arg()
m1
#> Error in eval(expr, envir, enclos): Argument "m1" fehlt (ohne Standardwert)
```

This gives it rather peculaiar behaviour since you can still access it in if it's stored in another data structure:


```r
m2 <- list(missing_arg())
m2[[1]]
```

If you do need to work with a missing argument stored in a variable, you can wrap up any accesses with `maybe_missing()` :


```r
maybe_missing(m1)
```

That prevents the error from occurring and instead returns another empty symbol.

You can see if it's misisng by using `is_missing()`:


```r
is_missing(m1)
#> [1] TRUE
is_missing(m2[[1]])
#> [1] TRUE
```

You only need to know about the missing symbol if you're programmatically creating functions with missing arguments; we'll come back to that in the next chapter. \index{names|empty}

### Calls

Calls define the tree in AST. A call behaves similarly to a list. It has a `length()`; you can extract elements with  `[[`, `[`, and `$`; and calls can contain other calls. The main difference is that the first element of a call is special: it's the function that will get called. \index{calls} Let's explore these ideas with a simple example:


```r
x <- expr(read.table("important.csv", row = FALSE))
lobstr::ast(!!x)
#> ??????read.table 
#> ??????"important.csv" 
#> ??????row = FALSE
```

The length of a call minus one gives the number of arguments:


```r
length(x) - 1
#> [1] 2
```

The names of a call are empty, except for named arguments:


```r
names(x)
#> [1] ""    ""    "row"
```

You can extract the leaves of the call by position and by name using `[[` and `$` in the usual way:


```r
x[[1]]
#> read.table
x[[2]]
#> [1] "important.csv"

x$row
#> [1] FALSE
```

Extracting specific arguments from calls is challenging because of R's flexible rules for argument matching: it could potentially be in any location, with the full name, with an abreviated name, or with no name. To work around this problem, you can use `rlang::lang_standardise()` which standardises all arguments to use the full name: \indexc{standardise\_call()}


```r
rlang::lang_standardise(x)
#> read.table(file = "important.csv", row.names = FALSE)
```

(Note that if the function uses `...` it's not possible to standardise all arguments.)

You can use `[` to extract multiple components, but if you drop the the first element, you're usually going to end up with a weird call:


```r
x[2:3]
#> "important.csv"(row = FALSE)
```

If you do want to extract multiple elements in this way, it's good practice to coerce the results to a list:


```r
as.list(x[2:3])
#> [[1]]
#> [1] "important.csv"
#> 
#> $row
#> [1] FALSE
```

Calls can be modified in the same way as lists:


```r
x$header <- TRUE
x
#> read.table("important.csv", row = FALSE, header = TRUE)
```

You can also construct a call from scratch using `rlang::lang()`. The first argument should be the function to be called (supplied either as a string or a symbol), and the subsequent arguments are the call to that function:


```r
lang("mean", x = expr(x), na.rm = TRUE)
#> mean(x = x, na.rm = TRUE)
```

### Pairlists

There is one data structure we need to discuss for completeness: the pairlist. \index{pairlists} Pairlists are a remnant of R's past and have been replaced by lists almost everywhere. The only place you are likely to see pairlists in R is when working with function arguments:


```r
f <- function(x = 10) x + 1
typeof(formals(f))
#> [1] "pairlist"
```

(If you're working in C, you'll encounter pairlists more often. For example, calls are also implemented using pairlists)
 
Fortunately, whenever you encounter a pairlist, you can treat it just like a regular list:


```r
pl <- pairlist(x = 1, y = 2)
length(pl)
#> [1] 2
pl$x
#> [1] 1
```

However, behind the scenes pairlists are implemented using a different data structure, a linked list instead of a vector. That means that subsetting is slower with pairlists, and gets slower the further along the pairlist you index:


```r
l1 <- as.list(1:20)
l2 <- as.pairlist(l1)

microbenchmark::microbenchmark(
  l1[[1]],
  l1[[10]],
  l2[[1]],
  l2[[10]]
)
#> Unit: nanoseconds
#>      expr min  lq mean median  uq  max neval cld
#>   l1[[1]] 186 198  268    210 234 4797   100  a 
#>  l1[[10]] 188 202  228    219 246  532   100  a 
#>   l2[[1]] 551 576  610    595 622  977   100   b
#>  l2[[10]] 554 586  656    604 632 4781   100   b
```

### Expression objects

Finally, we need to discuss the expression object briefly. \index{expression object}  Expression objects are produced by only two base functions: `expression()` and `parse()`:


```r
exp1 <- parse(text = c("
x <- 4
x
"))
exp2 <- expression(x <- 4, x)

typeof(exp1)
#> [1] "expression"
typeof(exp2)
#> [1] "expression"

exp1
#> expression(x <- 4, x)
exp2
#> expression(x <- 4, x)
```

\indexc{expression()} \indexc{parse()}

Like calls and pairlists, expression objects behave like a list:


```r
length(exp1)
#> [1] 2
exp1[[1]]
#> x <- 4
```

Conceptually an expression object is just a list of expressions. The only difference is that calling `eval()` on an expression evaluates each individual expression. We don't believe this advantage merits introducing a new data structure, so instead of expression objects we always use regular lists of expressions.

Because there might be many top-level calls in a file, `parse()` doesn't return just a single expression. Instead, it returns an expression object, which is essentially a list of expressions: 


```r
length(exp)
#> [1] 1
typeof(exp)
#> [1] "builtin"
```

### Exercises

1.  Which two of the six types of atomic vector can't appear in an expression? 
    Why? Why can't you create an expression that contains an atomic vector of 
    length greater than one? 

1.  `standardise_call()` doesn't work so well for the following calls.
    Why?

    
    ```r
    lang_standardise(quote(mean(1:10, na.rm = TRUE)))
    #> mean(x = 1:10, na.rm = TRUE)
    lang_standardise(quote(mean(n = T, 1:10)))
    #> mean(x = 1:10, n = T)
    lang_standardise(quote(mean(x = 1:10, , TRUE)))
    #> mean(x = 1:10, , TRUE)
    ```

1.  Why does this code not make sense?

    
    ```r
    x <- expr(foo(x = 1))
    names(x) <- c("x", "")
    ```

1.  `base::alist()` is useful for creating pairlists to be used for function
    arguments:
    
    
    ```r
    foo <- function() {}
    formals(foo) <- alist(x = , y = 1)
    foo
    #> function (x, y = 1) 
    #> {
    #> }
    ```
    
    What makes `alist()` special compared to `list()`?

1.  Concatenating a call and an expression with `c()` creates a list. Implement
    `concat()` so that the following code works to combine a call and
    an additional argument.

    
    ```r
    c(quote(f()), list(a = 1, b = quote(mean(a)))
    concat(quote(f), a = 1, b = quote(mean(a)))
    #> f(a = 1, b = mean(a))
    ```

## Case study: anaphoric functions

One useful application of `make_function()` is in functions like `curve()`. `curve()` allows you to plot a mathematical function without creating an explicit R function:


```r
curve(sin(exp(4 * x)), n = 1000)
```

<img src="Expressions_files/figure-html/curve-demo-1.png" width="70%" style="display: block; margin: auto;" />

Here `x` is a pronoun. `x` doesn't represent a single concrete value, but is instead a placeholder that varies over the range of the plot. One way to implement `curve()` would be with `make_function()`:


```r
curve4 <- function(expr, xlim = c(0, 1), n = 100) {
  expr <- enquo(expr)
  f <- new_function(alist(x = ), get_expr(expr), get_env(env))

  x <- seq(xlim[1], xlim[2], length = n)
  y <- f(x)

  plot(x, y, type = "l", ylab = deparse(substitute(expr)))
}
curve4(sin(exp(4 * x)), n = 1000)


curve3 <- function(expr, xlim = c(0, 1), n = 100) {
  expr <- enquo(expr)
  e <- rlang::expr({
    function(x) !!get_expr(expr)
  })
  f <- eval_tidy(e, env = get_env(expr))

  x <- seq(xlim[1], xlim[2], length = n)
  y <- f(x)

  plot(x, y, type = "l", ylab = deparse(substitute(expr)))
}

curve3(sin(exp(4 * x)), n = 1000)
```

<img src="Expressions_files/figure-html/curve2-1.png" width="70%" style="display: block; margin: auto;" />

Functions that use a pronoun are called [anaphoric](http://en.wikipedia.org/wiki/Anaphora_(linguistics)) functions. They are used in [Arc](http://www.arcfn.com/doc/anaphoric.html) (a lisp like language), [Perl](http://www.perlmonks.org/index.pl?node_id=666047), and [Clojure](http://amalloy.hubpages.com/hub/Unhygenic-anaphoric-Clojure-macros-for-fun-and-profit). \index{anaphoric functions} \index{functions!anaphoric}

### Exercises

1.  How are `alist(a)` and `alist(a = )` different? Think about both the
    input and the output.

1.  Read the documentation and source code for `pryr::partial()`. What does it
    do? How does it work? Read the documentation and source code for
    `pryr::unenclose()`. What does it do and how does it work?

1.  The actual implementation of `curve()` looks more like

    
    ```r
    curve3 <- function(expr, xlim = c(0, 1), n = 100,
                       env = parent.frame()) {
      env2 <- new.env(parent = env)
      env2$x <- seq(xlim[1], xlim[2], length = n)
    
      y <- eval(substitute(expr), env2)
      plot(env2$x, y, type = "l", 
        ylab = deparse(substitute(expr)))
    }
    ```

    How does this approach differ from `curve2()` defined above?


## Case study: Walking the AST with recursive functions {#ast-funs}

To conclude the chapter we're going to pull together everything that you've learned about ASTs to write solve more complicated tasks. Some inspiration comes from the base codetools package, which provides two interesting functions: \index{recursion!over ASTs}

* `findGlobals()` locates all global variables used by a function. This
  can be useful if you want to check that your function doesn't inadvertently
  rely on variables defined in their parent environment.

* `checkUsage()` checks for a range of common problems including
  unused local variables, unused parameters, and the use of partial
  argument matching.

Getting all of the details of this functions correct is fiddly and complex, so we're not going to explore the full expression. Instead we'll focus on the big ideas needed to implement functions that work like them, namely recursion. Recursive functions are a natural fit to tree-like data structures because a recursive function is made up of two parts:

* The __recursive case__ handles the nodes in the tree. Typically, you'll
  do something to each child of node, usually calling the recursive function
  again, and then combine the results back together again. For expressions, 
  you'll need to handle calls and pairlists in the recursive case.
  
* The __base case__ handles the leaves of the tree. The base cases ensure
  that the function eventually terminates, by solving the simplest cases 
  directly. For expressions, you need to base symbol and names in the base case.

To make this pattern easier to see, we'll use two helper functions. First we define `expr_type()` which will return "constant" for constant, "symbol" for symbols, "call", for calls, "pairlist" for pairlists, and the "type" of anything else:


```r
expr_type <- function(x) {
  if (rlang::is_syntactic_literal(x)) {
    "constant"
  } else if (is.symbol(x)) {
    "symbol"
  } else if (is.call(x)) {
    "call"
  } else if (is.pairlist(x)) {
    "pairlist"
  } else {
    typeof(x)
  }
}

expr_type(expr("a"))
#> [1] "constant"
expr_type(expr(f(1, 2)))
#> [1] "call"
```

We'll couple this with a wrapper around the switch function:


```r
switch_expr <- function(x, ...) {
  switch(expr_type(x), 
    ..., 
    stop("Don't know how to handle type ", typeof(x), call. = FALSE)  
  )
}
```

With these two functions in hand, the basic template for any function that walks the AST is as follows:


```r
recurse_call <- function(x) {
  switch_expr(x, 
    # Base cases
    symbol = ,
    constant = ,
    
    # Recursive cases
    call = ,
    pairlist = 
  )
}
```

Typically, solving the base case will easy so we'll do that first, and check the results. The recursive cases are a little more tricky, but typically you just need to think about the data structure you want in the output and then find the correct purrr function. To that end, make sure you're familiar with [Functionals] before continuing.

### Finding F and T

We'll start simple with a function that determines whether a function uses the logical abbreviations `T` and `F`: it will return `TRUE` if it finds a logical abbreviation, and `FALSE` otherwise. Using `T` and `F` is generally considered to be poor coding practice, and is something that `R CMD check` will warn about. 

Let's first compare the AST for `T` vs. `TRUE`:


```r
ast(TRUE)
#> TRUE
ast(T)
#> T
```

`TRUE` is parsed as a logical vector of length one, while `T` is parsed as a name. This tells us how to write our base cases for the recursive function: a constant is never a logical abbreviation, and a symbol is an abbreviation if it's "F" or "T":


```r
logical_abbr_rec <- function(x) {
  switch_expr(x, 
    constant = FALSE,
    symbol = as_string(x) %in% c("F", "T")
  )
}

logical_abbr_rec(expr(TRUE))
#> [1] FALSE
logical_abbr_rec(expr(T))
#> [1] TRUE
```

Our recursive function works with expressions which means we always need to use it with `expr()`. When writing a recursive function it's common to write a wrapper that provides defaults or makes the function a little easier to use. Here we'll typically make a wrapper that quotes its input (we'll learn more about that in the next chapter), so we don't need to use `expr()` every time.


```r
logical_abbr <- function(x) {
  logical_abbr_rec(enexpr(x))
}

logical_abbr(T)
#> [1] TRUE
logical_abbr(FALSE)
#> [1] FALSE
```

Next we need to implement the recursive cases. Here it's simple because we want to do the same thing for calls and for pairlists: recursively apply the function to each subcomponent, and return `TRUE` if any subcomponent contains a logical abbreviation. This is made easy by `purrr::some()`, which iterates over a list and returns `TRUE` if the predicate function is true for any element.


```r
logical_abbr_rec <- function(x) {
  switch_expr(x,
    # Base cases
    constant = FALSE,
    symbol = as_string(x) %in% c("F", "T"),
    
    # Recursive cases
    call = ,
    pairlist = purrr::some(x, logical_abbr_rec)
  )
}

logical_abbr(mean(x, na.rm = T))
#> [1] TRUE
logical_abbr(function(x, na.rm = T) FALSE)
#> [1] TRUE
```

### Finding all variables created by assignment

`logical_abbr()` is very simple: it only returns a single `TRUE` or `FALSE`. The next task, listing all variables created by assignment, is a little more complicated. We'll start simply, and then make the function progressively more rigorous. \indexc{find\_assign()} 

We start by looking at the AST for assignment:


```r
ast(x <- 10)
#> ??????`<-` 
#> ??????x 
#> ??????10
```

Assignment is a call where the first element is the symbol `<-`, the second is name of variable, and the third is the value to be assigned. 

Next, we need to decide what data structure we're going to use for the results. Here I think it will be easiest it we return a character vector. If we return symbols, we'll need to use a `list()` and that makes things a little more complicated.

With that in hand we can start by implementing the base cases and providing a helpful wrapper around the recursive function. The base cases here are really simple!


```r
find_assign_rec <- function(x) {
  switch_expr(x, 
    constant = ,
    symbol = character()
  )  
}
find_assign <- function(x) find_assign_rec(enexpr(x))

find_assign("x")
#> character(0)
find_assign(x)
#> character(0)
```

Next we implement the recursive cases. This is made easier by a function that should exist in purrrr, but currently doesn't:


```r
flat_map_chr <- function(.x, .f, ...) {
  purrr::flatten_chr(purrr::map(.x, .f, ...))
}
```

The recursive case for pairlists is simple: we iterate of every element of the pairlist (i.e. each function argument) and combine the results. The case for calls is a little bit more complex - if this is a call to `<-` then we should return the second element of the call:


```r
find_assign_rec <- function(x) {
  switch_expr(x,
    # Base cases
    constant = ,
    symbol = character(),
    
    # Recursive cases
    pairlist = flat_map_chr(as.list(x), find_assign_rec),
    call = {
      if (is_call(x, "<-")) {
        as_string(x[[2]])
      } else {
        flat_map_chr(as.list(x), find_assign_rec)
      }
    }
  )
}

# find_assign(function(x = 1) {})
# Need to handle srcrefs
# Maybe move to new section specifically about functions?

find_assign(a <- 1)
#> [1] "a"
find_assign({
  a <- 1
  {
    b <- 2
  }
})
#> [1] "a" "b"
```

Now we need to make our function more robust but coming up with examples intended to break it. What happens we assign to the same variable multiple times?


```r
find_assign({
  a <- 1
  a <- 2
})
#> [1] "a" "a"
```

It's easiest to fix this at the level of the wrapper function:


```r
find_assign <- function(x) unique(find_assign_rec(enexpr(x)))

find_assign({
  a <- 1
  a <- 2
})
#> [1] "a"
```

What happens if we have multiple calls to assign? Currently we only return the first. That's because when `<-` occurs we immediately terminate recursion. 


```r
find_assign({
  a <- b <- c <- 1
})
#> [1] "a"
```

Instead we need to take a more rigorous approach. I think it's best to keep the recursive function focused on the tree structure, so I'm going to extract out `find_assign_call()` into a separate function.


```r
find_assign_call <- function(x) {
  if (is_call(x, "<-") && is.symbol(x[[2]])) {
    lhs <- as_string(x[[2]])
    children <- as.list(x)[-1]
  } else {
    lhs <- character()
    children <- as.list(x)
  }
  
  c(lhs, flat_map_chr(children, find_assign_rec))
}

find_assign_rec <- function(x) {
  switch_expr(x,
    # Base cases
    constant = ,
    symbol = character(),
    
    # Recursive cases
    pairlist = flat_map_chr(x, find_assign_rec),
    call = find_assign_call(x)
  )
}

find_assign(a <- b <- c <- 1)
#> [1] "a" "b" "c"
find_assign(system.time(x <- print(y <- 5)))
#> [1] "x" "y"
find_assign(names(l) <- "b")
#> character(0)
```

While the complete version of this function is quite complicated, it's important to remember we wrote it by working our way up by writing simple component parts.

### Exercises

1.  `logical_abbr()` works with expressions. It currently fails when you give it
    a function. Why not? How could you modify `logical_abbr()` to make it 
    work? What components will you need to recurse over?

    
    ```r
    f <- function(x = TRUE) {
      g(x + T)
    }
    logical_abbr(!!f)
    ```

1.  Write a function that extracts all calls to a function.

1.  Instead of writing a wrapper that quotes its input, we could have made 
    the recursive function itself a quoting function. Why is this suboptimal?
