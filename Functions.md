# Functions

Functions are a fundamental building block of R: to master many of the more advanced techniques in this book, you need a solid foundation in how functions work. If you're reading this book, you've probably already created many R functions, and you're familiar with the basics of how they work. The focus of this chapter is to turn your existing, informal, knowledge of functions into a rigorous understanding of what functions are and how they work. You'll see some interesting tricks and techniques in this chapter, but most of what you'll learn is more important as building blocks for more advanced techniques.

Hopefully most of this chapter will be review and you can skim through it quickly, but you'll clarify your understand of functions as you go.

In this chapter you will learn:

* The three main components of a function.
* How scoping works, the process that looks up values from names.
* That everything in R is a function call, even if it doesn't look like it
* The three ways of supplying arguments to a function, and the impact of lazy evaluation.
* The two types of special functions: infix and replacement functions.

## Components of a function

There are three main components of a function:

* the `body()`, the code inside the function.

* the `formals()`, the "formal" arguments list, which controls how you can call the function.

* the `environment()` which determines how variables referred to inside the function are found.

When you print a function in R, it shows you these three important components. If the environment isn't displayed, it means that the function was created in the global environment. 

```R
f <- function(x) x
f

formals(f)
body(f)
environment(f)
```

There is an exception to this rule: primitive functions, like `sum`:

```R
sum
formals(sum)
body(sum)
environment(sum)
```

These are functions that call C code directly with `.Primitive()`. Primitive functions contain no R code and exist almost entirely in C, so their `formals()`, `body()` and `environment()` are all `NULL`. They are only found in the `base` package, and since they operate at a lower-level than most functions, they can be more efficient (primitive replacement functions don't have to make copies), and can have different rules for argument matching (e.g. `switch` and `call`). 

The assignment forms of `body()`, `formals()`, and `environment()` can also be used to modify functions. This is a useful technique which we'll explore in more detail in [[computing on the language]].

There are two other components that are possessed by some functions: the source of the function (which includes comments, unlike the `body()`), and if the function has been byte-code compiled, the byte code. 

### Exercises

* What function allows you to tell if a function is a primitive function or not?
* What are the three important components of a function?
* When does printing a function not show what environment it was created in?

## Lexical scoping

Scoping is the set of rules that govern how R looks up the value of a symbol, or name. That is, scoping is the set of rules that R applies to go from the symbol `x`, to its value `10` in the following example.

```R
x <- 10
x
# [1] 10
```

Understanding scoping allows you to:

* build tools by composing functions, as described in [[functional programming]]
* overrule the usual evaluation rules and [[compute on the language|computing-on-the-language]]

R has two types of scoping: __lexical scoping__, implemented automatically at the language level, and __dynamic scoping__, used in select functions to save typing during interactive analysis. We describe lexical scoping here because it is intimately tied to function creation. Dynamic scoping is described in the context of [[controlling evaluation|Evaluation]].

Lexical scoping looks up symbol values using how functions are nested when they were created, not how they are nested when they are called. With lexical scoping, you can figure out where the value of each variable will be looked up only by looking at the definition of the function, you don't need to know anything about how the function is called.

The "lexical" in lexical scoping doesn't correspond to the usual English definition ("of or relating to words or the vocabulary of a language as distinguished from its grammar and construction") but comes from the computer science term "lexing", which is part of the process that converts code represented as text to meaningful pieces that the programming language understands. It's lexical in this sense, because you only need the definition of the functions, not how they are called.

There are four basic principles behind R's implementation of lexical scoping:

* name masking
* functions vs. variables
* a fresh start
* dynamic lookup

You probably know some of these principles already - test your knowledge by mentally running the code in each block before looking at the answers.

### Name masking

The following example illustrates the simplest principle, and you should have no problem predicting the output.

```R
f <- function() { 
  x <- 1
  y <- 2
  c(x, y)
}
f()
rm(f)
```

If a name isn't defined inside a function, it will look one level up.

```R
x <- 2
g <- function() { 
  y <- 1
  c(x, y)
}
g()
rm(x, g)
```

The same rules apply if a function is defined inside another function.  First it looks inside the current function, then where that function was defined, and so on, all the way until the global environment. Run the following code in your head, then confirm the output by running the R code.

```R
x <- 1
h <- function() { 
  y <- 2
  i <- function() {
    z <- 3
    c(x, y, z)
  }
  i()
}
h()
rm(x, h)
```

The same rules apply to closures, functions that return functions. The following function, `j()`, returns a function.  What do you think this function will return when we call it?

```R
j <- function(x) {
  y <- 2
  function() {
    c(x, y)
  }
}
k <- j(1)
k()
rm(j, k)
```

This seems a little magical (how does R know what the value of `y` is after the function has been called), but it works because `k` keeps around the environment in which it was defined, which includes the value of `x`.  [[Environments]] gives some pointers on how you can dive in and figure out what some of the values are.

### Functions vs. variables

The same principles apply regardless of type of the associated value - finding functions works exactly the same way as finding variables:

```R
l <- function(x) x + 1
m <- function() {
  l <- function(x) x * 2
  l(10)
}
m()
rm(l, m)
```

There is one small tweak to the rule for functions. If you are using a name in a context where it's obvious that you want a function (e.g. `f(3)`), R will keep searching up the parent environments until it finds a function.  This means that in the following example `n` takes on a different value depending on whether R is looking for a function or a variable.

```R
n <- function(x) x / 2
o <- function() {
  n <- 10
  n(n)
}
o()
rm(n, o)
```

However, this can make for confusing code, and is generally best avoided.

### A fresh start

What happens to the values in between invocations of a function? What will happen the first time you run this function? What will happen the second time? (If you haven't seen `exists` before it returns `TRUE` if there's a variable of that name, otherwise it returns `FALSE`)

```R
j <- function() {
  if (!exists("a")) {  
    a <- 1
  } else {
    a <- a + 1
  }
  print(a)
}
```

You might be surprised that it returns the same value, `1`, every time. This is because every time a function is called, a new environment is created to host execution. A function has no way to tell what happened the last time it was run; each invocation is completely independent.

### Dynamic lookup

Lexical scoping determines where to look for values, not when to look for them. R looks for values when the function is run, not when it's created. This means results from a function can be different depending on objects outside its environment:

```R
f <- function() x
x <- 15
f()

x <- 20
f()
```

You generally want to avoid this behavour because it means the function is no longer self-contained. This is a common error - if you make a spelling mistake in your code, you won't get an error when you create the function, and you might not even get one when you run the function, depending on what variables are defined in the global environment. 

One way to detect this problem is the `findGlobals()` function from `codetools`. This function list all the external dependencies of a function:

```R
f <- function() x + 1
codetools::findGlobals(f)
```

Another way to try and solve the problem would be to manually change the environment of the function to the `emptyenv()`, an environment which contains absolutely nothing:

```R
environment(f) <- emptyenv()
f()
```

This doesn't work because R relies on lexical scoping to find _everything_, even the `+` operator.  

You can use this same idea to do other things that are extremely ill-advised. For example, since all of the standard operators in R are functions, you can override them with your own alternatives.  If you ever are feeling particularly evil, run the following code while your friend is away from their computer:

```R
"(" <- function(e1) {
  if (is.numeric(e1) && runif(1) < 0.1) {
    e1 + 1
  } else {
    e1
  }
}
replicate(100, (1 + 2))
rm("(")
```

This will introduce a particularly pernicious bug: 10% of the time, 1 will be added to any numeric operation carried out inside parentheses. This is yet another good reason to regularly restart with a clean R session!

### Exercises

* What does the following code return? Why? What does each of the three `c`'s mean?

  ```R
  c <- 10
  c(c = c)
  ```

* (From the R inferno 8.2.36): If `weirdFun()()()` is a valid command, what does `weirdFun()` return? Write an example.

* What does the following function return? Make a prediction before running the code yourself.

  ```R
  f <- function(x) {
    f <- function(x) {
      f <- function(x) {
        x ^ 2
      }
      f(x) + 1
    }
    f(x) * 2
  }
  f(10)
  ```

## Everything is a function call

The previous example of redefining `(` works because every operation in R is a function call, even operations with special syntax are just a disguise for ordinary function calls. This includes infix operators like `+`, control flow operators like `for`, `if`, and `while`, subsetting operators like `[]` and `$` and even the curly braces `{`. This means that each of these pairs of statements are exactly equivalent.  Note that `` ` ``, the backtick, lets you refer to functions or variables that have reserved or illegal names:

```R
x + y
`+`(x, y)

for (i in 1:10) print(i)
`for`(i, 1:10, print(i))

if (i == 1) print("yes!") else print("no.")
`if`(i==1, print("yes"), print("no."))

x[3]
`[`(x, 3)

{ print(1); print(2); print(3) }
`{`(print(1), print(2), print(3))
```

It is possible to override the definitions of these special functions, but almost certainly a bad idea. However, it can occassionally allow you to do something that would have otherwise been impossible. For example, this feature makes it possible for the `dplyr` package to translate R expressions into SQL expressions.

It's more often useful to treat special functions as ordinary functions. For example, we could use `lapply` to add 3 to every element of a list by first defining a function `add`, like this:

```R
add <- function(x, y) x + y
lapply(1:10, add, 3)
```

But we can get the same effect using the built in `+` function.

```R
lapply(1:10, `+`, 3)
lapply(1:10, "+", 3)
```

Note the difference between `` `+` `` and `"+"`.  The first one is the value of the object called `+`, and the second is a string containing the character `+`.  The second version works because `lapply` can be given the name of a function instead of the function itself.

That everything in R is represented as a function call will be more important to know for [[computing on the language]].

## Function arguments

It's useful to distinguish between the formal arguments and the actual arguments to a function. The formal arguments are a property of the function, whereas the actual or calling arguments vary each time you call the function. This section discusses how calling arguments are mapped to formal arguments, how default arguments work and the impact of lazy evaluation.

### Calling functions

When calling a function you can specify arguments by position, by complete name, or by partial name. Arguments are matched first by exact name, then by prefix matching and finally by position.

```R
f <- function(abcdef, bcde1, bcde2) {
  list(a = abcdef, b1 = bcde1, b2 = bcde2)
}
f(1, 2, 3)
f(2, 3, abcdef = 1)
# Can abbreviate long argument names:
f(2, 3, a = 1)
# Doesn't work because abbreviation is ambiguous
f(1, 3, b = 1)
```

Generally, you only want to use positional matching for the first one or two arguments: they will be the mostly commonly used, and most readers will probably know what they are. Avoid using positional matching for less commonly used arguments, and only use readable abbreviations with partial matching. (If you are writing code for a package that you want to publish on CRAN you can not use partial matching.) Named arguments should always come after unnamed arguments.

These are good calls:

```R
mean(1:10)
mean(1:10, trim = 0.05)
```

This is probably overkill:

```R
mean(x = 1:10)
```

And these are just confusing:

```R
mean(1:10, n = T)
mean(1:10, , FALSE)
mean(1:10, 0.05)
mean(, TRUE, x = c(1:10, NA))
```

### Default and missing arguments

Function arguments in R can have default values. 

```R
f <- function(a = 1, b = 2) {
  c(a, b)
}
f()
```

Since arguments in R are evaluated lazily (more on that below), the default value can be defined in terms of other arguments:

```R
g <- function(a = 1, b = a * 2) {
  c(a, b)
}
g()
g(10)
```

Default arguments can even be defined in terms of variables defined within the function. This is generally bad practice, because it makes it hard to understand what the default values will be without reading the complete source code of the function, and should be avoided.

```R
h <- function(a = 1, b = d) {
  d <- (a + 1) ^ 2
  c(a, b)
}
h()
h(10)
```

You can detect if an argument was supplied or not with the `missing()` function.

```R
i <- function(a, b) {
  c(missing(a), missing(b))
}
i()
i(a = 1)
i(b = 2)
i(1, 2)
```

However, I generally recommend against using `missing` because it makes it difficult to call programmatically from other functions (without using complicated workarounds). Generally, it's better to set a default value of `NULL` and then check with `is.null()`.

```R
j <- function(a = NULL, b = NULL) {
  c(is.null(a), is.null(b))
}
j()
j(a = 1)
j(b = 2)
j(1, 2)
```

### Lazy evaluation

By default, R function arguments are lazy - they're only evaluated if they're actually used:

```R
f <- function(x) {
  10
}
system.time(f(Sys.sleep(10)))
```

If you want to ensure that an argument is evaluated you can use `force`: 

```R   
f <- function(x) {
  force(x)
  10
}
system.time(f(Sys.sleep(10)))
```

This is important when creating closures with `lapply` or a loop:

```R
add <- function(x) {
  function(y) x + y
}
adders <- lapply(1:10, add)
adders[[1]](10)
adders[[10]](10)
```

`x` is lazily evaluated the first time that you call one of the adder functions. At this point, the loop is complete and the final value of `x` is 10.  Therefore all of the adder functions will add 10 on to their input, probably not what you wanted!  Manually forcing evaluation fixes the problem:

```R
add <- function(x) {
  force(x)
  function(y) x + y
}
adders2 <- lapply(1:10, add)
adders2[[1]](10)
adders2[[10]](10)
```

This code is exactly equivalent to


```R
add <- function(x) {
  x
  function(y) x + y
}
```

because the force function is just defined as `force <- function(x) x`. However, using this function serves as a clear indication that you're forcing evaluation, rather than having accidentally typed `x`.

Default arguments are evaluated inside the function. This means that if the expression depends on the current environment the results will be different depending on whether you use the default value or explicitly provide it.

```R
f <- function(x = ls()) {
  a <- 1
  x
}
# ls() evaluated inside f:
f()
# ls() evaluated in global environment:
f(ls())
```

More technically, an unevaluated argument is called a __promise__, or a thunk. A promise is made up of two parts:

* an expression giving the delayed computation, which can be accessed with `substitute` (see [[controlling evaluation|evaluation]] for more details)

* the environment where the expression was created and where it should be evaluated

You can find more information about a promise using `langr::promise_info`.  This uses some of R's C api to extract information about the promise without evaluating it (which is otherwise very tricky).

Laziness is useful in if statements - the second statement will be evaluated only if the first is true. (If it wasn't the statement would return an error because `NULL > 0` is a logical vector of length 0)

```R
x <- NULL
if (!is.null(x) && x > 0) {

}
```

We could implement "&&" ourselves:

```R
"&&" <- function(x, y) {
  if (!x) return(FALSE)
  if (!y) return(FALSE)

  TRUE
}
y <- NULL
!is.null(y) && y > 0
```

This function would not work without lazy evaluation because both `x` and `y` would always be evaluated, testing if `y > 0` even if `y` was NULL.

Sometimes you can also use laziness to elimate an if statement altogether. For example, instead of:

```R
if (is.null(y)) stop("Y is null")
```

You could write:
  
```R   
!is.null(y) || stop("Y is null")
```

Functions like `&&` and `||` have to be implemented as special cases in languages that don't support lazy evaluation because otherwise `x` and `y` are evaluated when you call the function, and `y` might be a statement that doesn't make sense unless `x` is true.

### `...`

There is a special argument called `...`.  This argument will match any arguments not otherwise matched, and can be used to call other functions.  This is useful if you want to collect arguments to call another function, but you don't want to prespecify their possible names.

To capture `...` in a form that is easier to work with, you can use `list(...)`. (See [[Computing on the language]] for other ways to capture ...)

Using `...` comes with a cost - any misspelled arguments will be silently ignored.  It's often better to be explicit rather than implicit, so you might instead ask users to supply a list of additional arguments.  And this is certainly easier if you're trying to use `...` with multiple additional functions.


## Special calls

R supports two additional syntaxes for calling functions you create: infix and replacement functions.

### Infix functions

Most functions in R are "prefix" operators: the name of the function comes before the arguments. You can also create infix functions where the function name comes in between its arguments, like `+` or `-`.  All infix functions names must start and end with `%` and R comes with the following infix functions predefined: `%%`, `%*%`, `%/%`, `%in%`, `%o%`,  `%x%`. 

(The complete list of built-in infix operators that don't need `%`is: `::, $, @, ^, *, /, +, -, >, >=, <, <=, ==, !=, !, &, &&, |, ||, ~, <-, <<-`)

For example, we could create a new operator that pastes together strings:

```R
"%+%" <- function(a, b) paste(a, b, sep = "")
"new" %+% " string"
```

Note that when creating the function, you have to put the name in quotes because it's a special name.

This is just a syntactic sugar for an ordinary function call; as far as R is concerned there is no difference between these two expressions:

```R
"new" %+% " string"
`%+%`("new", " string")
```

Or indeed between

```R
1 + 5
`+`(1, 5)
```

The names of infix functions are more flexible than regular R functions: they can contain any sequence of characters (except "%", of course). You will need to escape any special characters in the string used to define the function, but not when you call it:

```R
"% %" <- function(a, b) paste(a, b)
"%'%" <- function(a, b) paste(a, b)
"%/\\%" <- function(a, b) paste(a, b)

"a" % % "b"
"a" %'% "b"
"a" %/\% "b"
```

R's default precedence rules mean that infix operators are composed from left to right:

```R
"%-%" <- function(a, b) paste("(", a, " - ", b, ")", sep = "")
"a" %-% "b" %-% "c"
```

There's one infix function I've created that I use very often. It's inspired by Ruby's `||` logical or operator: it works a little differently to R's because of what objects ruby considers to be true, but it's often used as a way of setting default values.

```R
"%||%" <- function(a, b) if (!is.null(a)) a else b
function_that_might_return_null() %||% default value
```


### Replacement functions

Replacement functions act like they modify their arguments in place, and have the special name `xxx<-`. They typically have two arguments (`x` and `value`), although they can have more, and they must return the modified object. For example, the following function allows you to modify the second element of a vector:

```R
"second<-" <- function(x, value) {
  x[2] <- value
  x
}
x <- 1:10
second(x) <- 5L
x
```

When R evaluates the assignment `second(x) <- 5`, it notices that the left hand side of the `<-` is not a simple name, so it looks for a function named `second<-` to do the replacement.

If you want to supply additional arguments, they go in between `x` and `value`:

```R
"modify<-" <- function(x, position, value) {
  x[position] <- value
  x
}
modify(x, 1) <- 10
x
```

When you call `modify(x, 1) <- 10`, behind the scenes R turns it into:

```R
x <- `modify<-`(x, 1, 10)
```

This means you can't do things like:

```R
modify(get("x"), 1) <- 10`
```

because that gets turned into the invalid code:

```R
get("x") <- `modify<-`(get("x"), 1, 10)
```

It's often useful to combine replacement and subsetting, and this works out of the box:

```R
x <- c(a = 1, b = 2, c = 3)
names(x)
names(x)[2] <- "two"
names(x)
```

This works because the expression `names(x)[2] <- "two"` is evaluated as if you had written:

```R
`*tmp*` <- names(x)
`*tmp*`[2] <- "two"
names(x) <- `*tmp*`
```

(Yes, it really does create a local variable named `*tmp*`, which is removed afterwards.)

### Exercises

* 

## Return values

The last expression evaluated in a function becomes the return value, the result of invoking the function. 

```R
f <- function(x) {
  if (x < 10) {
    0
  } else {
    10
  }
}
f(5)
f(15)
```

Generally, I think it's good style to reserve the use of an explicit `return()` for when you are returning early, such as for an error, or a simple case of the function. This style of programming can also reduce the level of indentation, and generally make functions easier to understand because you can reason about them locally.

```R
f <- function(x, y) {
  if (!x) return(y)

  # complicated processing here
}
```

Functions can return only a single value, but this is not a limitation in practice because you can always return a list containing any number of objects.

The functions that are the most easy understand and reason about are pure functions, functions that always map the same input to the same output and have no other impact on the workspace. In other words, pure functions have no __side-effects__: they don't affect the state of the the world in anyway apart from the value they return. 

R protects you from one type of side-effect: arguments are passed-by-value, so modifying a function argument does not change the original value:

```R
f <- function(x) {
  x$a <- 2
  x
}
x <- list(a = 1)
f(x)
x$a
```

This is notably different to languages like Java where you can modify the inputs to a function. This copy-on-modify behaviour has important performance consequences which are discussed in depth in [[profiling]]. (Note that the performance consequences are a result of R's implementation of copy-on-modify semantics, they are not true in general. Clojure is a new language that makes extensive use of copy-on-modify semantics with limited performance consequences.)

Most base R functions are pure, with a few notable exceptions:

* `library` which loads a package, and hence modifies the search path

* `setwd`, `Sys.setenv`, `Sys.setlocale` which change the working directory, evnironment variables and the locale respectively

* `plot` and friends which produce graphical output

* `write`, `write.csv`, `saveRDS` etc which save output to disk

* `options` and `par` which modify global settings

* S4 related functions which modify global tables of classes and methods.

* random number generators which produce different numbers each time you run then

It's generally a good idea to minimise the use of side effects, and where possible separate functions into pure and impure, isolating side effects to the smallest possible location. Pure functions are easier to test (because all you need to worry about are the input values and the output), and are less likely to work differently on different versions of R or on different platforms.  For example, this is one of the motivating principles of ggplot2: most operations work on an object that represents a plot, and only the final `print` or `plot` call has the side effect of actually drawing the plot.

Functions can return `invisible` values, which are not printed out by default when you call the function.

```R
f1 <- function() 1
f2 <- function() invisible(1)

f1()
f2()
f1() == 1
f2() == 1
````

You can always force an invisible value to be displayed by wrapping it in parentheses:

```R
(f2())
```

The most common function that returns invisibly is `<-`:

```R
a <- 2
(a <- 2)
```

And this is what makes it possible to assign one value to multiple variables:

```R
a <- b <- c <- d <- 2
```

because that is parsed as:

```R
(a <- (b <- (c <- (d <- 2))))
```

## Deferred execution

**Caution**: Unfortunately the default in `on.exit()` is `add = FALSE`, so that every time you run it, it overwrites existing exit expressions.  Because of the way `on.exit()` is implemented, it's not possible to create a variant with `add = TRUE`, so you must be careful when using it.

### Exercises

*
