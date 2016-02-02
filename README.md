
# Introduction

Clojure is a simple language with a simple and elegant syntax.  This
style guide is also simple and primarily about how to format your
Clojure source.  To paraphrase the EmacsWiki:

> Clojure, Lisp, and Scheme have well established indentation rules.
> Don't break them.

# Guide

Since we read code far more often than we write it, it's important
that we learn how to read the code as well as write it.  This is
especially true in Clojure.  Consider this example:

``` clojure
(defn factorial [n]
  (if (< n 1)
      1
      (* n (factorial (dec n)))))
```

The novice might be tempted to count parentheses and brackets to see
what expressions are contained in other expressions, but that is the
wrong way to do it.  The experienced Clojure programmer will just
follow the indentation and assume that all the parens match up
perfectly.  The experienced reader will see that the `factorial`
function starts, has a single `if` expression, and the function ends
on the last line.  When the writer gets this wrong it is very easy for
the reader to draw the wrong conclusion about how the code works.

Therefor, the writer's job is to make sure that the code is easy to
read and structured in the way that the reader expects.

This guide will focus on how to structure and format code so it is
easy to read.  It will not get into the semantics of Clojure and tell
you when to use `seq`.  Instead it will focus on what column to start
code and how to write comments.  Except as noted below, use your
judgment on when to add line breaks.

## Bracket matching

Brackets match up.  A starting open paren will have a closing paren.
An open square bracket will have a matching close.  That is not
surprising, but, unlike many other languages, Clojure brackets are not
used to visually mark the end of a block.  Closing brackets almost
always are placed on the same line next to other brackets, with the
brackets all together and not on a line of their own.

The idea here is that a bracket on a line by itself or an extra line
break is just redundant information that doesn't help read the code.
Let's write our own `map` function called `xmap`:

``` clojure
(defn xmap [f coll]
  (lazy-seq
   (let [s (seq coll)]
     (when s
       (let [x (first s)
             xs (rest s)]
         (cons (f x) (xmap f xs)))))))
```

Notice how the indentation tells the story: the two-line `let` binding
followed by the body of the `let`.

There is an important exception that is often seen.  When a sequence
of items is written in code, the final closing brackets can be put on
their own line:


``` clojure
  :dependencies [
                 [cheshire "5.5.0"]
                 [clj-time "0.11.0"]
                 [com.cemerick/url "0.1.1"]
                 [ctdean/iter "0.10.1"]
                 [http-kit "2.1.19"]
                 [org.clojure/clojure "1.7.0"]
                 [org.clojure/java.jdbc "0.4.2"]
                 [org.clojure/tools.logging "0.3.1"]
                 [org.slf4j/slf4j-log4j12 "1.7.13"]
                 ]
```

## Indentation

Clojure indenting is very simple.  It's so simple that you might be
tempted to make your code more readable by adding some extra spaces to
help out the reader.  Resist this urge.  Readability will come by
indenting your code as everyone else does.

### Indent expressions equally

This is the basic rule.  Expressions are indented the same amount so
that each expression lines up.  They line up either with the first
argument to the enclosing expression or by one space if the first
argument is on a line by itself.  Like so:

``` clojure
(print "Algeria"
       "Bulgaria"
       "Cambodia"
       "Dominica"
       "Egypt"
       "France"
       "Gambia")
```

and

``` clojure
(print
 "Hungary"
 "Iran"
 "Japan"
 "Kazakhstan")
```

### Indent block expressions

Expressions that have a body argument where the sub-expressions are
evaluated in order, such as `let`, `do`, `defn`, or `fn`, we will call
_block expressions_.  Block expressions have the block portion
indented two spaces.  Like so:

```
(defn add1 [x]
  (+ 1 x))

(fn [x]
  (+ 1 x))

(let [a 1
      b "two"]
  (println "a => " a)
  (println "b => " b))
```

### Exceptions

There are a few popular exceptions to these rules (for example, the
`extend` form), but we're not going to talk about those forms.

## Line breaks between expressions

Put all sub-expressions on one line, or each sub-expression on a line
by itself.  As one of the Scheme guides says:

> Break for one - break for all.

``` clojure
(print :a :b :c)

(print :a
       :b
       :c)

(map vector
     (range)
     [:a :b :c :d :e])

```

## Comments

There are no completely agreed upon standards for formatting comments,
but these are in common usage and you are encouraged to follow them.

Clojure comments start with a semicolon and continue to the end of the
line.  The comment rules revolve around how many semicolons are used
and their starting column.

### Trailing Comments

A single semicolon comments a single line of code.  It starts on the
same line as the code:

``` clojure
(defn max [x y]
  (if (> x y)                           ; If x is bigger
      x                                 ; Return it
      y))                               ; Otherwise return y
```

### Block comments

Two semicolons are for commenting on a block of code.  The comment
begins immediately before the block, indented with the block, and
starts on a line by itself:

``` clojure
(defn factorial [n]
  ;; Remember that 0! is defined to be 1
  (if (< n 1)
      1
      (* n (factorial (dec n)))))
```

### Section comments

Three semicolons are for commenting a function or a section of a file.
Section comments start in the first column:

``` clojure
;;;
;;; Type Conversion
;;;

(defn string->symbol
  "Convert a string to a symbol"
  [s]
  (when s
    (symbol s)))

(defn string->integer
  "Convert a string to an integer."
  [s]
  (when s
    (Integer/parseInt s)))
```

If you're commenting a single function, consider using doc strings
instead.

### File header comments

In Clojure, instead of commenting a file, just use a namespace doc
string.  If you must comment a file, use four semicolons at the top of
the file.

## Naming

There are a few common naming conventions in Clojure.  We will attempt
to keep these brief.

### `foo?`

A predicate that returns a boolean.  Example: `(string? s)`

### `foo->bar`

Convert from type `foo` to type `bar`.  Example: `(string->integer s)`

### `with-foo`

Create a block where we clean up or close when the block ends.
Example:

``` clojure
(with-open [in (clojure.java.io/reader file-name)]
  (foo in))
```

# Examples

A few examples:

## Good

``` clojure
(cond
 (< x 0) "negative"
 (> x 0) "positive"
 :else "zero")
```

## Bad

Don't do these:

``` clojure
;; Too many args on a line
(+ 1 2
   x y z)

;; Extra indentation
(cond
 (< x 0)
   "negative"
 (> x 0)
   "positive"
 :else
   "zero")
```

# Links

- The scheme guide is very good:
  http://community.schemewiki.org/?scheme-style
- A popular Clojure style guide does discuss semantics:
  https://github.com/bbatsov/clojure-style-guide/blob/master/README.md

# Authors

Chris Dean <ctdean@sokitomi.com>
