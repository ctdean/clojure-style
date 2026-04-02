# Clojure Formatter Specification

Specification for building an executable that reads Clojure source
and writes formatted Clojure source to stdout.

## 1. Input and Output

The formatter reads one Clojure source file (from stdin or a file
argument) and writes the formatted result to stdout. Parse errors
cause the formatter to emit the input unchanged and exit with code 1.
Trailing whitespace is stripped from every output line.

## 2. Constants

- `MAX_LINE` = 90 characters
- `WIDE_PAIR_LIMIT` = 80 characters (threshold for splitting cond/case
  pairs onto separate lines)

## 3. Node Classification

The formatter operates on an AST produced by a Clojure parser (e.g.
rewrite-clj). Every node has a tag. The following tags are
**meaningful**: anything other than `:whitespace`, `:newline`, and
`:comment`. Non-meaningful nodes are skipped when counting arguments
or identifying operators.

### 3.1 Form Categories

Every list node `(...)` is classified by its first meaningful child
(the operator). The operator determines how the form is indented and
whether line merging is allowed.

**Block forms** receive 2-space body indentation. A form is a block
form if its operator symbol:

1. Appears in the block form set (Section 3.2), OR
2. Starts with the prefix `def`, OR
3. Starts with the prefix `define`, OR
4. Starts with the prefix `with-`

**Regular forms** receive argument-aligned indentation (wide or
narrow). Everything not classified as a block form is regular. In
particular, `if`, `if-not`, `cond`, `condp`, `case`, and all
threading macros (`->`, `->>`, `as->`, `some->`, `some->>`,
`cond->`, `cond->>`) are regular forms.

### 3.2 Block Form Set

```
def defn defn- defmacro defmulti defmethod defonce
defprotocol defrecord deftype defstruct definterface definline
let letfn binding locking fn bound-fn
with-open with-redefs with-local-vars with-bindings
with-in-str with-out-str with-precision with-loading-context
when when-not when-let when-some when-first if-let if-some while do
doseq dotimes doto for loop
future delay lazy-seq lazy-cat dosync sync io!
try catch finally
ns reify proxy comment time
extend-type extend-protocol extend gen-class gen-interface
deftest testing are with-test-out
go go-loop thread alt! alt!!
```

Additional symbols may be added via a `.clojure_format.edn` config
file (see Section 11).

### 3.3 No-merge Forms

These forms are never merged to a single line even if they would fit:

```
if if-not cond condp case
-> ->> as-> some-> some->> cond-> cond->>
```

### 3.4 Paired Argument Forms

These forms have arguments grouped in test/expression pairs:

- `cond`: 0 header args, all remaining are pairs
- `condp`: 2 header args, remaining are pairs
- `case`: 1 header arg, remaining are pairs

### 3.5 Binding Forms

These forms have a binding vector as their first vector child, where
entries are name/value pairs:

```
let letfn loop binding doseq dotimes for
if-let if-some when-let when-some when-first
```

### 3.6 Body Start Classification

For block forms, the body start position determines where
break-for-all enforcement begins. Arguments before the body start
are header arguments.

- **Position 0** (body at first arg after operator): `do`, `try`,
  `comment`, `future`, `delay`, `lazy-seq`, `lazy-cat`, `dosync`,
  `sync`, `io!`, `time`, `locking`, `go`, `thread`, `bound-fn`
- **Position 1** (body at second arg): `when`, `when-not`, `while`,
  `testing`, `finally`, `ns`
- **Position 2** (body at third arg): `catch`
- **After first vector** (body at first meaningful arg after the first
  vector child): all binding forms (Section 3.5), `fn`, `bound-fn`,
  and any symbol matching the `def*`, `define*`, or `with-*` prefix
  rules
- **Default for other block forms**: position 0

## 4. Indentation Rules

### 4.1 Regular Forms: Wide

When the first meaningful child after the operator appears on the
same line as the operator, all continuation lines align to the
starting column of that first argument.

```
column:  0         1
         0123456789012345
input:   (filter even?
                  ^--- target column for continuation lines
```

Algorithm:
1. Let `col` = starting column of the opening `(`
2. Find the first meaningful child (operator) and advance `col`
   past it
3. Skip whitespace (not newlines) after the operator
4. If the next meaningful child is on the same line, its column
   is `target_col`
5. Indent all subsequent continuation lines to `target_col`

### 4.2 Regular Forms: Narrow

When the first meaningful child after the operator is on a different
line (i.e., the operator stands alone on the first line), all
continuation lines align to `col + 1` where `col` is the column of
the opening `(`.

```
column:  0
         0123456789
input:   (filter
          ^--- target = col + 1 = 1
```

### 4.3 Block Forms

All continuation lines indent to `col + 2` where `col` is the column
of the opening `(`, regardless of where arguments appear on the first
line.

```
column:  2
         0123456789
input:     (when condition
             ^--- target = col + 2 = 4
```

### 4.4 Vectors, Maps, Sets, Anonymous Functions

These are not lists and have no operator. Continuation lines align
under the first meaningful element. The opening delimiter width is:

- Vector `[`: 1
- Map `{`: 1
- Set `#{`: 2
- Anonymous function `#(`: 2

If the first element is on its own line, indent to
`col + delimiter_width`.

### 4.5 Binding Vectors

Inside binding forms (Section 3.5), the first vector child contains
name/value pairs. These align under the first element of the vector,
following rule 4.4.

## 5. Line Break Rules

### 5.1 Break-for-all

If any meaningful argument in a form has a newline before it (i.e.,
it starts on a different line than the previous meaningful argument),
then ALL peer arguments at the same level must start on their own
lines.

**Scope**: break-for-all applies to body arguments only (see body
start classification, Section 3.6). Header arguments before the body
start are not subject to break-for-all.

**Multi-line arguments**: break-for-all examines where an argument
_starts_, not how many lines it spans. A multi-line argument (e.g., a
map literal) that starts on the same line as the previous argument
does not trigger break-for-all.

### 5.2 Paired Argument Forms

For paired argument forms (Section 3.4), break-for-all is replaced
by pair-aware formatting:

1. Each test argument starts a new line.
2. Each expression argument follows its test on the same line,
   separated by one space.
3. If any test/expression pair exceeds `WIDE_PAIR_LIMIT` characters:
   - Insert a blank line between pairs
   - Put each test and expression on their own lines

Both wide and narrow layouts are valid for `cond`:

```clojure
;; Narrow: cond stands alone, pairs at +1
(cond
 (< x 0) "Negative"
 (> x 0) "Positive"
 :else "Zero")

;; Wide: first pair on same line as cond
(cond (< x 0) "Negative"
      (> x 0) "Positive"
      :else "Zero")
```

### 5.3 Namespace Clauses

`:require`, `:import`, `:use`, and `:refer-clojure` clauses inside
`ns` forms are treated as block forms for break-for-all purposes.

## 6. Line Merging

When a form spans multiple lines but some arguments could be pulled
back onto the previous line, merge them if the result fits within
`MAX_LINE` characters.

### 6.1 Merge Algorithm

Process each form bottom-up. For each argument that starts on a new
line, check if its first-line width plus 1 (for a separating space)
plus the current column position is <= `MAX_LINE`. If so, replace
the newline and any indentation whitespace with a single space.

### 6.2 Do Not Merge

Skip merging entirely for:

- Block forms
- No-merge forms (Section 3.3)
- Maps and sets with 4 or more meaningful children (2+ key/value
  pairs)

Vectors are recursed into but their direct children are not merged.

## 7. Auto-break Long Lines

After indentation, any single-line form exceeding `MAX_LINE`
characters (including trailing closing delimiters from ancestor forms)
is broken.

### 7.1 Break Algorithm

1. Compute the total width: the form's text width at its starting
   column, plus the trailing width (closing delimiters of all
   ancestor forms that follow on the same line).
2. If total width > `MAX_LINE`:
   a. Walk the meaningful children in reverse order.
   b. For each candidate break point, compute the width of
      everything before it (at the current column).
   c. If that prefix width <= `MAX_LINE`, insert a newline before
      this child.
   d. Stop at the first (rightmost) candidate that fits.
3. If no candidate fits, leave the form unchanged.

For multi-line forms, recurse into each child and apply the same
check.

## 8. Binding Value Same-line

In binding forms (Section 3.5), the first vector child contains
name/value pairs. Each value must start on the same line as its name.

### 8.1 Algorithm

Walk the binding vector's meaningful children in alternating
name/value pairs. If a newline appears between a name and its value,
replace the newline and any indentation whitespace with a single
space.

## 9. Require Newline

Inside `:require` clauses, a newline must appear immediately after
the `:require` keyword. If whitespace (but not a newline) follows
`:require`, replace it with a newline. If a meaningful child follows
`:require` with no whitespace, insert a newline.

## 10. Bracket Placement

Closing delimiters (`)`, `]`, `}`) appear on the same line as the
last expression. A closing delimiter on a line by itself is
redundant and should be merged back.

**Exception**: for long sequences of items (`:require` lists,
dependency vectors), the final closing bracket may remain on its
own line.

## 11. Configuration

A `.clojure_format.edn` file may be placed in the current directory
or any ancestor directory. The formatter searches upward from the
current directory and uses the first file found.

Supported keys:

- `:block-forms` (set of symbols): additional symbols to treat as
  block forms, merged with the built-in set from Section 3.2.

## 12. Comments

Comments are passed through without reformatting. Their content is
never modified. The conventions for writing comments are:

- `;` : trailing comment, on the same line as code
- `;;` : block comment, on its own line, indented with surrounding
  code
- `;;;` : section comment, starts in column 0
- `;;;;` : file header comment, at the top of the file

## 13. Processing Pipeline

The formatter runs the following passes in order. The entire pipeline
executes **twice** for convergence (a second pass resolves cases
where one pass creates opportunities for another).

```
Pass 1: Line merging           (Section 6)
Pass 2: Binding value same-line (Section 8)
Pass 3: Require newline         (Section 9)
Pass 4: Break-for-all           (Section 5)
Pass 5: Indentation             (Section 4)
Pass 6: Auto-break long lines   (Section 7)
Pass 7: Indentation             (Section 4, again)
```

After both iterations, strip trailing whitespace from all lines.

## 14. Warnings

The formatter emits warnings to stderr (but does not fail) for:

- Any output line exceeding `MAX_LINE` characters (after all passes)
- `def`/`defonce` names containing hyphens (suggesting wrong naming
  convention)
- `defn`/`defn-`/`defmacro` names containing underscores (suggesting
  wrong naming convention)

## 15. Example

Input and expected output for a function exercising multiple rules:

```clojure
(defn read-file [path]
  (let [f (java.io.File. path)]
    (if (.exists f)
        (with-open [rdr (clojure.java.io/reader f)]
          (let [lines (doall (line-seq rdr))]
            (println (format "Read %d lines from %s"
                             (count lines)
                             (.getCanonicalPath f)))
            lines))
        nil)))
```

Breakdown:

| Form        | Category | Rule    | Indent  |
|-------------|----------|---------|---------|
| `defn`      | block    | 4.3     | col + 2 |
| `let`       | block    | 4.3     | col + 2 |
| `if`        | regular  | 4.1     | align under `(.exists f)` |
| `with-open` | block    | 4.3     | col + 2 |
| inner `let` | block    | 4.3     | col + 2 |
| `println`   | regular  | 4.1     | align under `(format ...)` |
| `format`    | regular  | 4.1 + 7 | align under `"Read..."`, auto-broken |
