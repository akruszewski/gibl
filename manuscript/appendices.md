# Appendices

## Appendix A: Cryptojacking

Modern Internet browsers are very powerful. They contain a built-in programming language called JavaScript that enables code execution on the viewer's computer. This is what enables web pages to be dynamic, or to do animations, etc.

As a consequence, when you visit a webpage on the Internet, this webpage can tell your browser's JavaScript to start mining.

I> ### Definition 1
I>
I> Cryptojacking is the unauthorized use of someone else's computer to mine blocks.

Recall that miners get reward (coins) for mining. In this case, your computer's resources are used for the purpose of mining. This serves as a motivation for cryptojacking.

TODO: Add some images

TODO: Maybe some example JavaScript code?

## Appendix B: Macros

So far we've been writing our definitions and interacted with them, but what if we had a way to write a piece of code that, when executed, would write a code itself? Macros are a way to do exactly that.

I> ### Definition 2
I>
I> A macro is a syntactic extension to a programming language. This means that we can write (or introduce our own) keywords in the programming language itself.

There is another special syntax in Lisp named `define-macro` which allows us to create a macro. It accepts a name of a macro, parameters (which are optional), and as a result it should return a quoted list of Lisp commands.

When we run our definitions (or compile them), Racket will replace all occurrences of the macro call with the actual code that we made it produce.

Macros are all about affecting how the evaluation model works. Racket has an eager evaluation model, which means that all expressions will be evaluated at a given time on a function call. For macros in contrast, since Racket will replace the code before evaluating it, expressions will be evaluated only when needed.

So, as an example, if we have the following macro `(define-macro (add-one x) (list '+ x 1))`, wherever we write `(add-one x)`, it will be replaced with the expression (+ x 1).

So, what is the difference between using a macro and a function? To answer, we will consider these definitions:

```racket
(require compatibility/defmacro)

(define-macro (our-if-macro a b c)
  (list 'cond (list a b) (list 'else c)))

(define (our-if-function a b c)
  (cond (a b) (else c)))
```

With a few evaluations:

```racket
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

The way we've written our macro makes things much more explicit in terms of execution and substitution. We can also write it as follows, which is a bit more implicit:

```racket
(define-macro (our-if-macro a b c)
  `(cond (,a ,b) (else ,c)))
```

### Hygienic macros

The way we've written our macros earlier are not a good practice in Racket. To see why, consider the following example:

```racket
(define-macro (swap a b)
  `(let ((tmp ,a))
     (set! ,a ,b)
     (set! ,b tmp)))
```

This looks like a perfectly safe macro. Indeed:

```racket
> (define x 1)
> (define y 2)
> (displayln (list x y))
(1 2)
> (swap x y)
> (displayln (list x y))
(2 1)
```

However, we can break it by using the following code:

```racket
> (define x 1)
> (define tmp 2)
> (displayln (list x tmp))
(1 2)
> (swap x tmp)
> (displayln (list x tmp))
(1 2)
```

To see what happened, we can expand the macro by hand. The macro translates to:

```racket
> `(let ((tmp ,x))
    (set! ,x ,tmp)
    (set! ,tmp tmp))
'(let ((tmp 2)) (set! 2 2) (set! 2 tmp))
```

As we can see, it is hard to control accidental capture of local identifiers. We can get around the problem using `gensym` which returns a unique symbol every time it's called:

```racket
(define-macro (swap a b)
  (let ((tmp (gensym)))
    `(let ((,tmp ,a))
       (set! ,a ,b)
       (set! ,b ,tmp))))
```

Now our macro works as intended:

```racket
> (define x 1)
> (define tmp 2)
> (displayln (list x tmp))
(1 2)
> (swap x tmp)
> (displayln (list x tmp))
(1 2)
```

However, instead of relying on `gensym`, the preferred way in Racket is to use `define-syntax-rules`:

```racket
(define-syntax-rule (swap a b)
  (let ((tmp a))
    (set! a b)
    (set! b tmp)))
```

## Appendix C: Text editor

Writing is important, so we will write a text editor. Cool!

Explain the struct

```racket
(struct ed (buffer position) #:prefab)
```

We will define evaluations. Start by defining the procedure:

```racket
(define (eval-ed e c)
  (match c
```

We define line reading:

```racket
    [`n (list (ed-position e)
              (list-ref (ed-buffer e) (sub1 (ed-position e))))]
    [`p (string-join (ed-buffer e) "\n")]
```

Line movement:

```racket
    [`+ (let ([next-pos (add1 (ed-position e))])
          (ed (ed-buffer e) (if (<= next-pos (length (ed-buffer e))) next-pos (ed-position e))))]
    [`- (let ([prev-pos (sub1 (ed-position e))])
          (ed (ed-buffer e) (if (> prev-pos 0) prev-pos (ed-position e))))]
    [(? number? new-pos)
     (ed (ed-buffer e) (if (and (> new-pos 0)
                                (<= new-pos (length (ed-buffer e))))
                           new-pos
                           (ed-position e)))]
```

We now define insertion and deletion:

```racket
    [`(i ,str)
     (ed (insert-at (ed-buffer e) (ed-position e) str) (add1 (ed-position e)))]
    [`d (ed (remove-at (ed-buffer e) (ed-position e))
            (if (= (ed-position e) (length (ed-buffer e)))
                (sub1 (length (ed-buffer e)))
                (ed-position e)))]
```

Line copying:

```racket
    [`(t ,from ,to)
     (if (not (and (> from 0)
                   (<= from (length (ed-buffer e)))
                   (> to 0)
                   (<= to (length (ed-buffer e)))))
         (ed (ed-buffer e) (ed-position e))
         (ed (insert-at (ed-buffer e) to (list-ref (ed-buffer e) (sub1 from))) (add1 to)))]
```

And finally, all other cases:

```racket
    [_ error "Unknown command"]))
```

Together with these helper procedures:

```racket
(define (insert-at lst pos x)
  (define-values (before after) (split-at lst pos))
  (append before (cons x after)))

(define (remove-at lst pos)
  (append (take lst (sub1 pos))
          (drop lst pos)))

(define (make-ed buffer)
  (ed buffer (length buffer)))
```

We will now interact with our editor. Start by creating a "file" and printing line:

```racket
> (define sample-file (make-ed '("line one" "line two" "line three")))
> (eval-ed sample-file 'n)
'(3 "line three")
```

Test the line movement:

```racket
> (set! sample-file (eval-ed sample-file '-))
> (eval-ed sample-file 'n)
'(2 "line two")
> (set! sample-file (eval-ed sample-file '2))
> (eval-ed sample-file 'n)
'(2 "line two")
> (set! sample-file (eval-ed sample-file '+))
> (eval-ed sample-file 'n)
'(3 "line three")
```

Test insertion, together with printing the whole "file":

```racket
> (set! sample-file (eval-ed sample-file '(i "line four")))
> (eval-ed sample-file 'n)
'(4 "line four")
> (eval-ed sample-file 'p)
"line one\nline two\nline three\nline four"
```

We test deletion next:

```racket
> (set! sample-file (eval-ed sample-file '2))
> (set! sample-file (eval-ed sample-file 'd))
> (eval-ed sample-file 'n)
'(2 "line three")
> (eval-ed sample-file 'p)
"line one\nline three\nline four"
```

And finally, line copying:

```racket
> (set! sample-file (eval-ed sample-file '(t 1 2)))
> (eval-ed sample-file 'n)
'(3 "line one")
> (eval-ed sample-file 'p)
"line one\nline three\nline one\nline four"
```
