# Appendices

## Appendix A: Macros

So far we've been writing our definitions and interacted with them, but what if we had a way to write a piece of code that, when executed, would write a code itself? Macros are a way to do exactly that.

There is another special syntax in Lisp named `define-macro` which allows us to create a macro. It accepts a name of a macro, parameters (which are optional), and as a result it should return a quoted list of Lisp commands.

When we run our definitions (or compile them), Racket will replace all occurrences of the macro call with the actual code that we made it produce.

Macros are all about affecting how the evaluation model works. Racket has an eager evaluation model, which means that all expressions will be evaluated at a given time on a function call. For macros in contrast, since Racket will replace the code before evaluating it, expressions will be evaluated only when needed.

So, as an example, if we have the following macro `(define-macro (add-one x) (list '+ x 1))`, wherever we write `(add-one x)`, it will be replaced with the expression (+ x 1).

So, what is the difference between using a macro and a function? To answer, we will consider these definitions:

```
(require mzlib/defmacro)

(define-macro (our-if-macro a b c) (list 'cond (list a b) (list 'else c)))
(define (our-if-function a b c) (cond (a b) (else c)))
```

With a few evaluations:

```
> (our-if-macro (eq? '() '()) #t #f)
#t
> (our-if-function (eq? '() '()) #t #f)
#t
> (our-if-macro (eq? '() '()) (display "True") (display "False"))
True
> (our-if-function (eq? '() '()) (display "True") (display "False"))
TrueFalse
```

We can notice a couple of things from the code above:

1. We required a library that contains the `define-macro` syntax
1. We've implemented our own `if` both as a macro and as a function
1. In the interactions area we've used a function called `display`, which prints stuff to the output
1. We see that the macro and the function produce two different outputs

The macro and the function behave differently, as expected. However, in the case of `if`, it makes sense implementing it as a macro rather than a function. It does not make sense to evaluate the `else` case if we are sure that the first case will match.
