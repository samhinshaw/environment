# Environments

Reference: http://adv-r.had.co.nz/Environments.html

If you're like me, when you hear "environment", you think of the place where all your assigned variables and functions live--and where you can see them in your RStudio Environment tab.

However, if we think about this for a bit, we realize that it's not so simple!  You can't just name variables anything, what if you...
```r
str <- 24
```
Is `str` still available as a function? Of course it is! But how is that possible? Well, you may already know the answer, and it is *multiple environments*. You know about this already from the bedeviling "bugs" in your code when you use `filter()` instead of `dplyr::filter()`. But how does this all work? Let's find out!

## Topics:

1. Environment Basics
2. Functions
3. Packages
4. Back to Basics


*Note: You may want to install the `pryr` package before we begin today.*






## Environment Basics

Open up R(Studio) and enter `search()`. As you can immediately see, there is what we know as the "Global Environment", ".GlobalEnv", and an environment for each package currently loaded, including the {base} packages. When you call a function, R searches these environments in order.

![Hadley's Example](http://adv-r.had.co.nz/diagrams/environments.png/search-path.png)

Aside: Setting an object to NULL does not remove it, but sets it to nothing. So `rm(object)` will remove it, and you can check its existence with `exists()`. BUT, here's when things get tricky! If you're working in a specific environment and you delete an object within that environment, R will look through parent environments with `exists()` by default.  To see what I mean, let's create a new environment.

```r
NewEnviron <- new.env()
NewEnviron$object <- 27
```

Now if we just wanted to create an object within the global environment (what you've probably been doing up until now), we would just use:
```r
object <- 27
```

SO, if you've run both of these commands and then want to remove the object:
```r
rm(NewEnviron$object)
```

If you check for its existence, it will still be there!
```r
exists("x", envir = e)
#> [1] TRUE
```
It searched in the parent environments as well. To specify NOT to do this, we can provide an argument:
```r
exists("x", envir = e, inherits = FALSE)
#> [1] FALSE
```

## Functions

An *enclosing* environment of a function is the environment where the function was created. Every function has one and only one enclosing environment. For the three other types of environment, there may be 0, 1, or many environments associated with each function:

  - Binding a function to a name with `<-` defines a *binding* environment.
  - Calling a function creates an ephemeral *execution* environment that stores variables created during execution.
  - Every execution environment is associated with a *calling* environment, which tells you where the function was called.

### Creating Functions

This image sums up a basic function creation. You create a function with the name "f"! It's done in the global environment, so that's where the name of the function lives.

![Function Creation](http://adv-r.had.co.nz/diagrams/environments.png/binding.png)

This part is important. WHERE you create the function determines WHERE the function searches for values! Even if you move the function, assign it a name in a different environment, it will look for values in the environment it was created in.

![Functions Searching](http://adv-r.had.co.nz/diagrams/environments.png/binding-2.png)

Hadley explains why this matters:

>The distinction between the binding environment and the enclosing environment is important for package namespaces. Package namespaces keep packages independent. For example, if package A uses the base mean() function, what happens if package B creates its own mean() function? Namespaces ensure that package A continues to use the base mean() function, and that package A is not affected by package B (unless explicitly asked for).

So it seems that our multiplicity of environments arises mostly from packages. We'll come back to that in a second.

### Running functions

>Each time a function is called, a new environment is created to host execution. The parent of the execution environment is the enclosing environment of the function.

This intuitively makes sense, right? We already knew that functions executed in somewhat of a sandbox, but that sandbox is still linked to the outside world via its "parent".

Here is a diagram of a function running!

![Run That Function](http://adv-r.had.co.nz/diagrams/environments.png/execution.png)


## Packages

Let's start with Hadley's example from a base R package.

```r
environment(sd)
#> <environment: namespace:stats>
where("sd")
#> <environment: package:stats>
```

The definition of sd() uses var(), but if we make our own version of var() it doesn’t affect sd():


```r
x <- 1:10
sd(x)
#> [1] 3.02765
var <- function(x, na.rm = TRUE) 100
sd(x)
#> [1] 3.02765
```

This deceptively simple concept is illustrated in this complex figure!

![Complex Complexity](http://adv-r.had.co.nz/diagrams/environments.png/namespace.png)

Reminder: example from a NAMESPACE document (from [jennybc/googlesheets](https://github.com/jennybc/googlesheets))
```r
export(gs_key)
importFrom(dplyr,"%>%")
```





## Back to Basics

### Assignment

>Assignment is the act of binding (or rebinding) a name to a value in an environment. You’ve probably used regular assignment in R thousands of times. Regular assignment creates a binding between a name and an object in the current environment. Names usually consist of letters, digits, . and _, and can’t begin with _. If you try to use a name that doesn’t follow these rules, you get an error:
```r
_abc <- 1
# Error: unexpected input in "_"
```
Reserved words (like TRUE, NULL, if, and function) follow the rules but are reserved by R for other purposes:
```r
if <- 10
#> Error: unexpected assignment in "if <-"
```

>A complete list of reserved words can be found in `?Reserved`.  It’s possible to override the usual rules and use a name with any sequence of characters by surrounding the name with backticks:
```r
`a + b` <- 3
`:)` <- "smile"
`    ` <- "spaces"
ls()
#  [1] "    "   ":)"     "a + b"
`:)`
#  [1] "smile"
```

>You can also create non-syntactic bindings using single and double quotes instead of backticks, but I don’t recommend it. The ability to use strings on the left hand side of the assignment arrow is a historical artefact, used before R supported backticks.

### Deep Assignment

> The regular assignment arrow, `<-`, always creates a variable in the current environment. The deep assignment arrow, `<<-`, never creates a variable in the current environment, but instead modifies an existing variable found by walking up the parent environments. You can also do deep binding with `assign(): name <<- value` is equivalent to `assign("name", value, inherits = TRUE)`.

```r
x <- 0
f <- function() {
  x <<- 1
}
f()
x
#> [1] 1
```

>If `<<-` doesn’t find an existing variable, it will create one in the global environment. This is usually undesirable, because global variables introduce non-obvious dependencies between functions. `<<-` is most often used in conjunction with a closure, as described in [Closures](http://adv-r.had.co.nz/Functional-programming.html#closures).

### Explicit Environments

Fun fact, to modify an environment you do not assign it to anything! Simply calling the function will modify it!

*But Why??*

**Avoiding copies**

- Since environments have reference semantics, you’ll never accidentally create a copy. This makes it a useful vessel for large objects. It’s a common technique for bioconductor packages which often have to manage large genomic objects. Changes to R 3.1.0 have made this use substantially less important because modifying a list no longer makes a deep copy. Previously, modifying a single element of a list would cause every element to be copied, an expensive operation if some elements are large. Now, modifying a list efficiently reuses existing vectors, saving much time.

**Package state**

- Explicit environments are useful in packages because they allow you to maintain state across function calls. Normally, objects in a package are locked, so you can’t modify them directly. Instead, you can do something like this:

**As a hashmap**

- A hashmap is a data structure that takes constant, O(1), time to find an object based on its name. Environments provide this behaviour by default, so can be used to simulate a hashmap. See the CRAN package hash for a complete development of this idea.

*Reminder*: The parent of the global environment is the last package that you loaded. The only environment that doesn’t have a parent is the empty environment.
