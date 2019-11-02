# 2. Racket programming language

## 2.1. Introduction to Lisp

Lisp, originating from 1958, stands for LISt Processing. Unlike standard programming languages, it has a fully parenthesized prefix notation. For example, instead of writing `1 + 2`, one would write `(+ 1 2)`.

There are three important notions in a Lisp:

1. Primitives or axioms, starting points. As an example, the numbers 1, 2, etc. are something we do not have to implement ourselves since they are already included in the programming language. Another example is operations upon the numbers, such as `+`, `*`, etc.
1. Composition or a way to compose primitives to do complex calculations. For example we can combine `+` and `*` as follows: `1 + (2 * 3)` or in prefix notation: `(+ 1 (* 2 3))`
1. Abstraction or capturing composition of primitives. For example, if we find ourselves doing some calculation over and over again, it would be good to capture (abstract, or wrap) it in a function that can be easily re-used

I> ### Definition 1
I>
I> A data structure is a collection of values, the relationships among them, and the functions or operations that can be applied to the data.

As we have mentioned, an example of a data structure is numbers together with the plus and multiplication functions.

From the motivation in the previous section we can see a need of forming such a composite data type, where, for example, a block is a structure that contains a hash, an owner, transaction amount, etc.

There are many data structures. An ordered list is one example, representing the numbers {$$}(1, 2, 3){/$$} in that order. There are operations on lists, such as counting number of elements, appending two lists, etc.

I> ### Definition 2
I>
I> In mathematics and computer science, objects exhibit recursive behavior when they can be defined by two properties:
I>
I> 1. A simple base case (or cases) - a terminating case that returns a value without using recursion
I> 1. A set of rules that reduce all other cases toward the base case

The best example is the factorial function, defined as:

{$$}fact(n) = \left\{ \begin{array}{ll} 1\text{, if } n = 0 \\	n * fact(n - 1)\text{, otherwise} \end{array} \right.{/$$}

For example, using only substitution we can see that {$$}fact(3){/$$} evaluates to {$$}3 \cdot fact(2){/$$} which is {$$}3 \cdot 2 \cdot fact(1){/$$}, and then finally {$$}3 \cdot 2 \cdot 1 \cdot fact(0){/$$} which is just 6.

I> ### Definition 3
I>
I> A tree is a hierarchical, recursive data structure that can have two possible values:
I>
I> 1. An empty value
I> 1. A single value, coupled together with another two sub-trees

TODO: Example of a tree.

I> ### Definition
I>
I> A **language** is consisted of:
I>
I> 1. _Symbols_, which can be combined into sentences
I> 1. _Grammar_, which is a set of rules that tells us which sentences are well-formeed

The definition of a language also reflects programming languages - they have a special syntax and reserved keywords. For example, the C programming language has keywords such as `struct`, `return`, `if`, etc.

I> ### Definition 4
I>
I> An abstract syntax tree is a tree representation of the abstract syntactic structure of a source code written in a programming language.

When you write a program in a programming language, there's an intermediate step that parses the program's source code and derives an abstract syntax tree.

![An abstract syntax tree](images/ast.png)

For example, the image above represents an abstract syntax tree for the following code in C:

```c
while (x > 0) {
    x = x - 1;
    y = y * 2;
}
```

It is not important to understand what this code does, rather how the compiler represents such code internally.

Lisps have no syntax in the way that most standard languages have. What we will write as code is the actual abstract syntax tree. This is why Lisps rely on prefix notation. Thus, Lisps are based on a minimalistic design, so we do not get the overhead of many other languages that have special keywords, where sometimes some functionalities overlap with existing ones.

TODO: The word syntax has special meaning in Lisps. In Lisps, there's a thing called "reader", which is part of the process when we execute a program. The "reader" reads in the source code, then turns that into a sequence of lisp primitives such as lists, symbols, numbers, etc. This, lispers think of as “the lisp syntax”. With macros as part of the core language it's possible to build new syntax[^ch2n1].

The Racket programming language that we will use in this book is a multi-paradigm programming language, belonging to the Lisp family.

## 2.2. Why Racket

Racket (formerly known as PLT Scheme) is a Lisp. It's not just Lisp, rather a Lisp, since there are many Lisp implementations, but we found this one to be particularly easy for entry level programmers.

The language is used in a variety of contexts such as scripting, general-purpose programming, computer science education, and research. It has been used for commercial projects. One notable example is the Hacker News website, which runs on Arc, a programming language developed in Racket. Racket is also used to teach students algebra through game development.

Scheme, the programming language from which Racket was influenced and based upon, was created in the 1970s at the MIT by Guy L. Steele and Gerald Jay Sussman. Scheme is widely used by a number of schools, as a programming language in introductory courses for computer science.

TODO: Move before? In our opinion, building a cryptocurrency (or anything, for that matter) in Racket will imply that you can do the same in most other languages with ease. This programming language favors function composition, and we will see further in the book the interesting properties that composition offers and how easily we can update and refactor our code.

There are two main approaches to work with Racket:

1. Using the graphical user interface (GUI), which is the recommended way and the way that we will use throughout this book
1. Using the command line utilities (`racket` - the interpreter/compiler, `raco` - the package manager, etc) for more advanced users

## 2.3. Configuration and installation

Racket can be downloaded and installed via https://download.racket-lang.org. There are available binaries for Windows, Linux, and Mac. After having downloaded and installed the complete package, we can run DrRacket. If you get the following screen after trying to run it, congratulations! It means that the installation was successful.

![DrRacket](images/drracket.png)

The upper textarea part is the definitions area, where we usually write our definitions. Alternatively, the lower part is the interactions area where we interact with the definitions.

I> ### Definition
I>
I> A package in Racket resembles a set of definitions someone has written for others to use.

For example, if we want to use a hashing function, we will include the package for hashing in order to have the hashing definitions available to interact with. This allows us to put our focus on the design of our system, instead of re-defining everything.

Packages can be browsed at http://pkgs.racket-lang.org. Packages can be installed from the DrRacket GUI - when we try to use a package that is missing and available in the packages repository, DrRacket will give us the option to install it. Alternatively, they can be installed using `raco pkg install <package_name>` from the command line. We will take advantage of packages in Racket later in the book.

The Help Desk under `Help > Help Desk` contains useful information such as quick introduction, reference manuals, examples, etc. which is also available in offline-mode, but is optional for this book.

## 2.4. Tutorial

The first thing that Lisp newcomers notice is that there are too many parentheses in most Lisp programs. This is true, but it is a direct consequence of that we are actually writing our own abstract syntax tree in a language that has no special syntax.

As we go through this book, we will understand the power of expressiveness we get as a result. For example, one advantage is that there is no need for special order of operations. In high school, we had to remember that `*` and `/` have to come before `+` and `-`. This is not the case with Lisps, as the order of evaluation is obvious by the way we've written our program.

So, let's start by writing `(+ 1 (* 2 3))`, followed by the return key, in the interactions area of the DrRacket editor:

```racket
> (+ 1 (* 2 3))
7
```

If you get 7 as a result, then congratulations! You have done your first calculation in Racket.

After it finished evaluation, DrRacket again waits for us to input a new command. This is so because in the interactions area we are in the REPL mode, which stands for Read-Evaluate-Print-Loop. That is, interactions area will read what we write, try to evaluate it (come up with a result), print the result, and loop back to reading again.

Lisp evaluation is very similar to substitution in mathematics. For example, one way `(+ 1 (* 2 3))` can be evaluated is as follows:

1. `(+ 1 (* 2 3))`
1. `(+ 1 6)`
1. `7`

We immediately notice how powerful substitution as a concept is.

### 2.4.1. Primitive types

In the evaluation we've done above, what we get as a result is a number. So the value 7 has a type of number. While this may be implicit in Racket, we have a way to check what the type of a value is, as we will see later with the help of predicates.

Racket has some primitive types, such as: numbers, booleans, strings, lists, and functions.

```racket
> 123
123
> #t
#t
> #f
#f
> "Hello World"
"Hello World"
```

Each of the evaluations above have a specific type attached to the value produced:

1. The first evaluation has a type of number
1. The second evaluation (which stands for true) has a type of boolean
1. The third evaluation (which stands for false) has a type of boolean
1. The fourth evaluation has a type of a string

### 2.4.2. Lists, evaluation, quotes

In order to produce the ordered list {$$}(1, 2, 3){/$$}, we can ask DrRacket to evaluate `(list 1 2 3)`:

```racket
> (list 1 2 3)
'(1 2 3)
```

`list` is a built-in function, just like `+`. `list` accepts any number of parameters, and as a result returns a list generated from them.

We notice how parentheses are used to denote a function call, or evaluation. In general, the code `(f a_1 a_2 ... a_n)` makes a function call to `f`, passing `n` parameters in that order. For example, for the function {$$}f(x) = x + 1{/$$}, one example evaluation is {$$}f(1){/$$} where as a return value we get 2.

However, now, as a result, we get `'(1 2 3)`. Let's try to understand what happened here. If it had returned `(1 2 3)`, this wouldn't have made much sense, since as we discussed above this notation would try to call the function 1 with arguments 2 and 3. Instead, it returned a *quoted* list. This is the same as saying `'(1 2 3)`.

To understand how this affects the evaluation model better, let's consider an example where you say either of these statements to some friend of yours:

1. Say your name
1. Say "your name"

In the first example, you expect your friend to tell you their name. However, in the second example, you expect them to say "your name", rather than their actual name.

There is a built-in syntax called `quote`. So the expression `'(1 2 3)` is just a fancy notation which is equivalent to the expression `(quote (1 2 3))`, where we tell Racket to return the actual list `(1 2 3)`, instead of evaluating it.

There is a special list, called the empty list and is denoted as `'()` or `(quote ())`. We will later see why this list is special when we'll talk about recursion.

### 2.4.3. Pairs

Another built-in function is `cons` which stands for construct. This function only accepts two parameters, and as a result it returns a pair:

```racket
> (cons 1 2)
'(1 . 2)
```

![The `(cons some-value nil)` pair](images/pair.png)

There are two other built-in functions called `car` and `cdr` which are used to retrieve the first and the second element of a pair respectively:

```racket
> (car (cons 1 2))
1
> (cdr (cons 1 2))
2
```

Pairs are so important that we can encode any data structure with them. In fact, lists are a special kind of pairs, where `(list 1 2 3)` is equal to `(cons 1 (cons 2 (cons 3 '())))`.

![An example of a list](images/list.png)

The motivation for using a list is that it will allow us, for exeample, to link several blocks together to make a blockchain.

Finally, we notice how in Lisp, depending only on a few primitive notions (function calls, pairs, `quote`) we can build abstraction in an interesting way.

### 2.4.4. Adding definitions

So far, we have only worked with the interactions area in DrRacket. Let's try to do something useful with the definitions area.

![Adding definitions in DrRacket](images/drracket-definitions.png)

We can notice a couple of things from the screenshot above:

1. In the definitions area, we added some code. We notice that we used another built-in syntax called `define` in order to attach a value (`123`) to a symbol (`a-number`)
1. In the interactions area, we interacted with something that was already defined in the definitions area (in this case, the interaction was to just display its value)

In this book every Racket program will start with `#lang racket`. This means that we will be dealing with Racket's ordinary syntax. There are other values this can accept, for example we can work with a language specialized in drawing graphics, but that is out of context for the purpose of this book.

Everything that we type in the definitions area, we can also type in the interactions area and vice-versa. In order to have the definitions available in the interactions area we need to Run the program by navigating to the top menu `Racket > Run`. Note that when we Run a program, the interactions area gets cleared.

If our definitions have references, using the mouse we can hover over the symbol's name and DrRacket will draw a line pointing to the definition of that reference. For big and complex programs, it is advised to use this approach to fully understand how one function works, when necessary.

![Following references in DrRacket](images/drracket-refs.png)

Definitions can be saved to a file for later usage by navigating to `File > Save Definitions`.

### 2.4.5. Procedures and functions

In Lisp, a procedure is essentially a function. When invoked, it returns some data as its value. We will use the words "procedure" and "function" interchangeably.

Some Lisp expressions and procedures have side effects, for example, doing a network operation, which means that this function can return different values at different points in time. Thus Lisp procedures are not always functions in the "pure" sense of mathematics, but in practice they are frequently referred to as "functions" anyway, even those that may have side effects, in order to emphasize that a computed result is always returned.

There is a special built-in syntax called `lambda`, which accepts two parameters, and produces a function for us as a result. The first parameter is a list of arguments this functions accepts, and the second parameter is an expression that acts upon these parameters.

For example, `(lambda (x) (+ x 1))` returns a function that accepts a single parameter, and when this function is called with a parameter, it increases this parameter's value by one.

If we try to evaluate the expression above, we get:

```racket
> (lambda (x) (+ x 1))
#<procedure>
```

So, in order to call our function, we can try to pass a parameter to its return value:

```racket
> ((lambda (x) (+ x 1)) 1)
2
```

Of course, writing functions this way is hard. Instead, what we can do is define our function in the definitions area and then interact with it in the interactions area:

Definition:

```racket
(define add-one (lambda (x) (+ x 1)))
```

Interaction:

```racket
> (add-one 1)
2
> (add-one 2)
3
> (add-one (add-one 1))
3
```

To make things a little bit easier for us, Racket has a special syntax for defining functions, so these two are equivalent:

```racket
(define add-one (lambda (x) (+ x 1)))
(define (add-one x) (+ x 1))
```

### 2.4.6. Comparison functions

There are some very useful functions that produce boolean output for us, such as checking whether a number is greater than another one, or whether a value is a number. We can notice the usage of some of them in the code below:

```racket
(define (add-one x) (+ x 1))
(define x 1)
(define hello "Hello World")
```

Interacting with it

```racket
> (number? x)
#t
> (number? hello)
#f
> (string? hello)
#t
> (procedure? add-one)
#t
> (> 1 2)
#f
> (= 1 2)
#f
> (= 1 1)
#t
```

We can also do a conditional check, and evaluate expressions based on the truthiness of some predicate (comparison function).

For example, `if` is a built-in syntax that accepts three parameters:

1. Conditional to check
1. Expression to evaluate if the conditional is true
1. Expression to evaluate if the conditional is false

For example, `(if (= 1 1) "It is true" "It is not true")` will return `"It is true"`, whereas `(if (= 1 2) "It is true" "It is not true")` will return `"It is not true"`.

The more general syntax for `if` is `cond`, which has the following syntax:

```racket
(cond (test-1 action-1)
   (test-2 action-2)
   ...
   (test-n action-n))
```

Optionally, the last test can be an else to use the specific action if none of the conditions above match.

As an example, here is one way to interact with it:

```racket
(define (is-large x)
  (cond ((> x 10) #t)
        (else #f)))
```

Interacting

```racket
> (is-large 5)
#f
> (is-large 10)
#f
> (is-large 11)
#t
```

As we've seen, the `=` is an equivalence predicate used to check whether two numbers are equal. However, it works only on numbers and it will raise an error if we use it on anything else:

```racket
> (= 1 1)
#t
> (= 2.5 2.5)
#t
> (= '() '())
=: contract violation
```

There are two important predicates which we will take a look at, and these are `eq?` and `equal?`. The `eq?` predicate is used to check whether its two parameters represent the same object in memory.

As an example:

```racket
> (define x '(1 2))
> (define y '(1 2))
> (eq? x y)
#f
> (define y x)
> (eq? x y)
#t
```

Note, however that there's only one empty list `'()` in memory (actually the empty list doesn't exist in memory, but a pointer to the memory location 0 is considered as the empty list).

So, when comparing empty lists, `eq?` will always return #t because they represent the same object in memory. Note that different type comparisons with eq? depend on the implementation of a Lisp.

The `equal?` predicate is exactly the same as the `eq?` predicate, except that it can be used on primitive types (numbers, strings, lists) to check if they are equivalent. For example:

```racket
> (define x '(2 3))
> (define y '(2 3))
> (equal? x y)
#t
```

### 2.4.7. Recursive procedures

Procedures can also be recursive, which means that we can call the procedure within itself in attempt to make a computation, or a loop.

For example, here is a demo which defines a function that calculates a length of a list, and how we interact with it:

```racket
(define (list-length x)
  (cond ((equal? x '()) 0)
        (else (+ 1 (list-length (cdr x))))))
```

Running

```racket
> (list-length '(1 2 3))
3
> (list-length '())
0
> (list-length '(1))
1
```

Let's try to understand how this example works.

First, we define a function `list-length` that accepts a single parameter `x`, and in the body of the function we have a condition:

1. If we are passing an empty list, just return 0 since the length of an empty list is 0
1. Otherwise, return the value of `(list-length (cdr x))` plus one

Note how we discussed that list are a special type of a pair:

```racket
> (car '(1 2 3))
1
> (cdr '(1 2 3))
'(2 3)
> (car (cdr '(1 2 3)))
2
> (cdr (cdr '(1 2 3)))
'(3)
```

What we can notice from the example above is that the `cdr` of a list will return that same list without the first element.

So here is how Racket evaluates `(list-length '(1 2 3))`:

```racket
(list-length '(1 2 3))
(+ 1 (list-length '(2 3)))
(+ 1 (+ 1 (list-length '(3))))
(+ 1 (+ 1 (+ 1 (list-length '()))))
(+ 1 (+ 1 (+ 1 0)))
(+ 1 (+ 1 1))
(+ 1 2)
3
```

So we see how this exhibits a recursive behaviour since the recursive cases were reduced to the base case in attempt to get a result.

With this example, we can see the power of recursion and how it allows us to do processing of values in a repeating manner.

A recursive procedure can generate an iterative or a recursive process.

An iterative process is a process where the state is captured completely by its arguments. A recursive one, in contrast is one where the state is not captured by the arguments, and so it relies on "deferred" evaluations.

In the example above, list-length generates a recursive process since it needs to go down to the base case, and then build its way back up to do the calculations that were "deferred".

In contrast, we can re-write `list-length` as:

```racket
> (define (list-length-iter x n)
  (cond ((equal? x '()) n)
        (else (list-length-iter (cdr x) (+ n 1)))))
> (list-length-iter '(1 2 3) 0)
3
```

This procedure now generates an iterative process, since the results are captured in the arguments.

So here is how Racket evaluates `(list-length-iter '(1 2 3) 0)`:

```racket
(list-length-iter '(1 2 3) 0)
(list-length-iter '(2 3) 1)
(list-length-iter '(3) 2)
(list-length-iter '() 3)
3
```

### 2.4.8. Procedures that return procedures

We can also construct functions that return other functions as a result. For example:

```racket
> (define (f x) (lambda (y) (+ x y)))
> f
#<procedure:f>
> (f 1)
#<procedure>
> ((f 1) 2)
3
```

This concept is so powerful, that we can implement our own `cons`, `car`, and `cdr`:

```racket
(define (my-cons x y) (lambda (z) (if (= z 1) x y)))
(define (my-car x) (x 1))
(define (my-cdr x) (x 2))
```

Evaluating

```racket
> (my-cons 1 2)
#<procedure>
> (my-car (my-cons 1 2))
1
> (my-cdr (my-cons 1 2))
2
```

To explain what happened here, note how we define `my-cons` to return another function that accepts a parameter, and then based on that parameter we either return the first parameter to `my-cons` or the second one.

If we try to use the substitution method, we can note that `(my-cons 10 20)` evaluates to `(lambda (z) (if (= z 1) 10 20))`. So our function "captures" data in a sense.

Then, when we call `my-car` or `my-cdr` on this function, we just pass 1 or 2 to get the first or the second value respectively.

### 2.4.9. General higher order procedures

With the example above we've seen how Lisp can return a function as a return value. It can also accept a function as an input.

In general, a higher order procedure is one that takes one or more functions as arguments, or returns a function as its result.

There are three general built-in higher order procedures: `map`, `filter`, `fold` (left and right).

As an example usage:

Definitions

```racket
(define my-test-list '(1 2 3))
(define (add-one x) (+ x 1))
```

Interact

```racket
> (map add-one my-test-list)
'(2 3 4)
> (filter even? my-test-list)
'(2)
> (foldr cons '() '(1 2 3))
'(1 2 3)
> (foldl cons '() '(1 2 3))
'(3 2 1)
```

Note in the example above how we evaluated expressions in the definitions area itself and get the produced results in the interactions area.

Here's a description of each higher order function used in the above example:

1. `map` is a function that takes as input a function with a single parameter, and returns a list where all members of the list have this function applied
1. `filter` is a function that takes as input a function (predicate) with a single parameter (that returns a boolean), and only returns those members in the list whose function evaluates to true
1. `fold` is a function that takes as input a combining function that accepts two parameters, an initial value and returns a list combined with this fun

There are two types of folds, a left and a right one, which combines from the left and from the right respectively.

For example, the right fold may use `(+ 1 (+ 2 (+ 3 0)))`, while the left one `(+ (+ (+ 0 1) 2) 3)`.

So, if we use the substitution method on `(map add-one my-test-list)`, we get: `(list (add-one 1) (add-one 2) (add-one 3))`.

We can actually implement these functions ourselves:

```racket
(define (my-map f l)
  (cond ((eq? l '()) '())
        (else (cons (f (car l)) (my-map f (cdr l))))))
(define (my-filter p l)
  (cond ((eq? l '()) '())
        ((p (car l)) (cons (car l) (my-filter p (cdr l))))
        (else (my-filter p (cdr l)))))
```

It is recommended that you attempt to try to implement the folding functions yourself. Note that `foldl` generates an iterative process, while `foldr` generates a recursive one.

### 2.4.10. Packages

In computer science, a library is a collection of resources used by computer programs. These may include configuration data, documentation, help data, message templates, pre-written code and procedures, structures, or values. Racket packages are similar to a library.

To import a library, we use the syntax `(require <library_name>)`.

As an example, let's create a few procedures, and then save their definitions in a file called `utils.rkt` by clicking on `File > Save Definitions`:

`utils.rkt`:

```racket
(define (sum-list l) (foldl + 0 l))
(define (add-one x) (+ x 1))
(provide sum-list)
```

Now if we create another file called `test.rkt` in the same folder where `utils.rkt` is, and we write the following code:

```racket
> (require "utils.rkt")
> (sum-list '(1 2 3))
6
> (add-one 1)
add-one: undefined;
```

We can notice how only the functions we provide within the special syntax `(provide ...)` will be available for usage by those who require our library.

### 2.4.11. Scope

Let's consider the following definitions:

```racket
(define my-number 123)
(define (add-to-my-number x) (+ my-number x))
```

Here we can notice a couple of things:

1. We created a variable `my-number` and assigned the number 123 to it
1. We created a function `add-to-my-number` which adds the number passed to it as a parameter to `my-number`

One question is, depending on the current state of execution, what is the visibility of our variables?

Scope refers to the visibility of the definitions, or which parts of the program can use them.

For example, we know that `my-number` is in the scope of `add-to-my-number`, and so we can conclude that `my-number` is kind of a global variable accessible anywhere within the definitions list.

But what about the `x` within `add-to-my-number`? This variable is only accessible within the body of the function definition, and no longer accessible outside it.

We can assign a new value to already defined variables by using the set! syntax as follows:

```racket
(define my-number 123)
(define (add-to-my-number x) (+ my-number x))
(add-to-my-number 1)
(set! my-number 100)
(add-to-my-number 2)
```

This program will first output 124, and then 102. Note how our function keeps referring to the global variable `my-number`, and so may produce different results depending on its value.

With `let` we can define variables within the current scope that it is executed. The syntax is:

```racket
(let ([var-1 value-1]
      [var-2 value-2])
  ... our code ...)
```

This creates "temporary" variables `var-1` and `var-2` visible only in the "our code" part, and after it is executed they are no longer accessible from outside the scope.

```racket
> (let ([x 1] [y 2]) (+ x y))
3
> x
. . x: undefined;
> y
. . y: undefined;
```

There is another syntax `letrec` which is very similar to `let`, in that it allows variables to be visible in the variable scope as well, so we can do:

```racket
> (letrec ([x 1] [y 2] [z (+ x y)]) z)
3
```

Note that `[` and `]` have the same function as `(` and `)`, they are just used for clearer display and meaning of the code.

Another interesting syntax is `for` and `for*` which have similar meaning to `let` and `letrec`. We can see how `for*` generates a Cartessian set in the following screenshot:

```racket
(displayln "for")

(for ([i (range 1 3)]
      [j (range 1 3)])
  (displayln (cons i j)))

(displayln "for*")
(for* ([i (range 1 3)]
      [j (range 1 3)])
  (displayln (cons i j)))
```

Produces

```racket
for
(1 . 1)
(2 . 2)
for *
(1 . 1)
(1 . 2)
(2 . 1)
(2 . 2)
```

## Summary

TODO: See what we need to move from here

In Racket, there's a special syntax (that is, a macro) named `define-struct` which allows us to capture data structures and come up with a new kind of abstraction.

In a sense, we already know how we can capture abstractions with `car`, `cons`, and `cdr`, however `define-struct` is much more convenient since once we defined our data structures it will automatically provide procedures for us to construct such data type and retrieve its values.

A few examples:

```racket
> (struct document (author title content))
> (struct book document (publisher))
```

Having entered the first command, we automatically get the procedures `document-author`, `document-title`, `document-content` in order to extract values from objects, and the procedure document in order to construct object of such type. Now, we can construct an object that is using this data structure:

```racket
> (define a-book
>   (document
>    "Boro Sitnikovski"
>    "Gentle Introduction to Blockchain with Lisp"
>    "Hello World"))
```

We can also use the automatically generated procedures to extract values from objects that are using this data structure:

```racket
> (document-author a-book)
"Boro Sitnikovski"
> (document-title a-book)
"Gentle Introduction to Blockchain with Lisp"
> (document-content a-book)
"Hello World"
```

Throughout this book, we will use `serializable-struct` (from the `racket/serialize` library) instead of `struct`, since this will allow for serializing data structures and writing them to the file system for example.

A structure in most programming languages is a composite data type (or record) declaration that defines a physically grouped list of variables to be placed under one name in a block of memory.

[^ch2n1]: More on macros in Appendix A.
