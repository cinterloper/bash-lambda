# Bash lambda

Real lambda support for bash (a functionally complete hack). Includes a set of
functions for functional programming, list allocation and traversal, and
concurrent mark/sweep garbage collection with weak reference support.

## NOTE

*This library is experimental.* Don't use it for mission-critical applications,
important data storage, etc. It's obviously a huge hack and may malfunction in
any number of awkward ways. None of these should impact the integrity of the
rest of your data (I source this library in my `.bashrc` and nothing bad has
happened yet), but it probably wouldn't hurt to glance over the source code
before using it, just in case.

## Getting started

Load the library into your shell (the `source` line can go into .bashrc):
```
$ git clone git://github.com/spencertipping/bash-lambda
$ source bash-lambda/bash-lambda
```

## Defining functions

The functions provided by bash-lambda take their nomenclature from Clojure, and
they are as analogous as I could make them while remaining somewhat useful in a
bash context. They are, by example:

### Anonymous functions

These are created with the `fn` form and work like this:

```
$ $(fn name 'echo "hello there, $name"') spencer
hello there, spencer
$ greet=$(fn name 'echo "hello there, $name"')
$ $greet spencer
hello there, spencer
$
```

The spec is `fn formals... 'body'`. You can use an alternative form, `cons_fn`,
which takes the body from stdin: `cons_fn formals... <<'end'\n body \nend`.

### Named functions

These are created using `defn`:

```
$ defn greet name 'echo "hi there, $name"'
/tmp/blheap-xxxx-xxxxxxxxxxxxxxxxxxx/greet
$ greet spencer
hi there, spencer
$
```

Notice that we didn't need to dereference `greet` this time, since it's now a
named function instead of a variable.

You can use `def` to name any value:

```
$ def increment $(fn x 'echo $((x + 1))')
$ increment 10
11
$
```

### Partials and composition

Like Clojure, bash-lambda gives you functions to create partial (curried)
functions and compositions. These are called `partial` and `comp`, respectively.

```
$ add=$(fn x y 'echo $((x + y))')
$ $(partial $add 5) 6
11
$ $(comp $(partial $add 1) $(partial $add 2)) 5
8
$
```

## Lists

Bash-lambda gives you the usual suspects for list manipulation. However, there
are a few differences from the way they are normally implemented.

### Mapping over lists

```
$ map $(partial $add 5) $(list 1 2 3 4)
6
7
8
9
$ seq 1 4 | map $(partial $add 1)
2
3
4
5
$
```

The `list` function boxes up a series of values into a single file for later
use. It's worth noting that this won't work:

```
$ map $(partial $add 5) $(map $(partial $add 1) $(list 1 2 3))
cat: 2: No such file or directory
$
```

You need to wrap the inner `map` into a list if you want to use applicative
notation:

```
$ map $(partial $add 5) $(list $(map $(partial $add 1) $(list 1 2 3)))
7
8
9
$
```

Alternatively, just use pipes. This allows you to process lists of arbitrary
size.

```
$ map $(partial $add 1) $(list 1 2 3) | map $(partial $add 5)
7
8
9
$
```

### Reducing and filtering

Two functions `reduce` and `filter` do what you would expect. (`reduce` isn't
named `fold` like the Clojure function because `fold` is a useful shell utility
already)

```
$ reduce $add 0 $(list 1 2 3)
6
$ even=$(fn x '((x % 2 == 0))')
$ seq 1 10 | filter $even
2
4
6
8
10
$
```

Higher-order functions work like you would expect:

```
$ sum_list=$(partial reduce $add 0)
$ $sum_list $(list 1 2 3)
6
$ rand_int=$(fn 'echo $RANDOM')
$ repeatedly $rand_int 100 | $sum_list
1566864

$ our_numbers=$(list $(repeatedly $rand_int 100))
```

### Flatmap (mapcat in Clojure)

It turns out that `map` already does what you need to write `mapcat`. The lists
in bash-lambda behave more like Perl lists than like Lisp lists -- that is,
consing is associative, so things are flattened out unless you box them up into
files. Therefore, `mapcat` is just a matter of writing multiple values from a
function:

```
$ self_and_one_more=$(fn x 'echo $x; echo $((x + 1))')
$ map $self_and_one_more $(list 1 2 3)
1
2
2
3
3
4
$
```

### Infinite lists

UNIX has a long tradition of using pipes to form lazy computations, and
bash-lambda is designed to do the same thing. You can generate infinite lists
using `iterate` and `repeatedly`, each of which is roughly equivalent to the
Clojure function of the same name:

```
$ repeatedly $rand_int 100      # notice count is after, not before, function
<100 numbers>
$ iterate $(partial $add 1) 0
0
1
2
3
...
$
```

Note that both `iterate` and `repeatedly` will continue forever, even if you
use `take` (a wrapper for `head`) to select only a few lines. I'm not sure how
to fix this at the moment, but I'd be very happy to accept a fix if anyone has
one. (See `src/list` for these functions)

### Searching lists

Bash-lambda provides `some` and `every` to find list values. These behave like
Clojure's `some` and `every`, but each one returns the element it used to
terminate the loop, if any.

```
$ ls /bin/* | some $(fn '[[ -x $1 ]]')
/bin/bash
$ echo $?
0
$ ls /etc/* | every $(fn '[[ -d $1 ]]')
/etc/adduser.conf
$ echo $?
1
$
```

If `some` or `every` reaches the end of the list, then it outputs nothing and
its only result is its status code. (For `some`, it means nothing was found, so
it returns 1; for `every`, it means they all satisfied the predicate, so it
returns 0.)

## Closures

There are two ways you can allocate closures, one of which is true to the usual
Lisp way but is horrendously ugly:

```
$ average=$(fn xs "echo \$((\$($sum_list \$xs) / \$(wc -l < \$xs)))")
$ $average $our_numbers
14927
```

Here we're closing over the current value of `$sum_list` and emulating
Lisp-style quasiquoting by deliberately escaping everything else. (Well, with a
good bit of bash mixed in.)

The easier way is to make the 'sum_list' function visible within closures by
giving it a name. While we're at it, let's do the same for `average`.

```
$ def sum-list $sum_list
$ defn average xs 'echo $(($(sum-list $xs) / $(wc -l < $xs)))'
```

Named functions don't need to be dereferenced, since they aren't variables:

```
$ average $our_numbers
14927
```

Values are just files, so you can save one for later:

```
$ cp $our_numbers the-list
```

## Garbage collection

Bash-lambda implements a conservative concurrent mark-sweep garbage collector
that runs automatically if an allocation is made more than 30 seconds since the
last GC run. This prevents the disk from accumulating tons of unused files from
anonymous functions, partials, compositions, etc.

You can also trigger a synchronous GC by running the `bash_lambda_gc` function
(just `gc` for short):

```
$ bash_lambda_gc
0 0
$
```

The two numbers are the number and size of reclaimed objects. If it says `0 0`,
then no garbage was found.

### Inspecting heap state

You can see how much memory the heap is using by running `heap_stats`:

```
$ heap_stats
heap size:           120K
objects:             54
permanent:           54
$
```

You can also inspect the root set and find the references for any object:

```
$ gc_roots | gc_refs
/tmp/blheap-xxxx-xxxxxxxxxxx/gc_roots
/tmp/blheap-xxxx-xxxxxxxxxxx/gc_roots
/tmp/blheap-xxxx-xxxxxxxxxxx/gc_roots
$ add=$(fn x y 'echo $((x + y))')
$ echo $add
/tmp/blheap-xxxx-xxxxxxxxxxx/fn_xxxx_xxxxxxxxx
$ cat $(partial $add 5) | gc_refs
/tmp/blheap-xxxx-xxxxxxxxxxx/fn_xxxx_xxxxxxxxx
$
```

The output of `gc_refs` includes weak references. You can detect weak or
fictitious references by looking for slashes in whatever follows
`$BASH_LAMBDA_HEAP/`:

```
$ is_real_ref=$(fn x '[[ ! ${x##$BASH_LAMBDA_HEAP/} =~ / ]]')
$ $is_real_ref $is_real_ref || echo not real
$ $is_real_ref $(weak_ref $is_real_ref) || echo not real
not real
$
```

### Pinning objects

The root set is built from all variables you have declared in your shell. This
includes any functions you've written, etc. However, there may be cases where
you need to pin a reference so that it will never be collected. You can do this
using `bash_lambda_gc_pin`, or just `gc_pin` for short:

```
$ pinned_function=$(gc_pin $(fn x 'echo $x'))
```

### Weak references

You can construct a weak reference to anything in the heap using `weak_ref`:

```
$ f=$(fn x 'echo hi')
$ g=$(weak_ref $f)
$ $f
hi
$ $g
hi
$ unset f
$ $g
hi
$ bash_lambda_gc
1 36
$ $g
no such file or directory
$
```

Weak references (and all references, for that matter) can be checked using
bash's `-e` test:

```
$ f=$(fn x 'echo hi')
$ exists=$(fn name '[[ -e $name ]] && echo yes || echo no')
$ $exists $f
yes
$ g=$(weak_ref $f)
$ $exists $g
yes
$ unset f
$ bash_lambda_gc
1 36
$ $exists $g
no
$
```

### Limits of concurrent GC in bash

Bash-lambda doesn't own its heap and memory space the same way that the JVM
does. As a result, there are a few cases where GC will be inaccurate, causing
objects to be collected when they shouldn't. So far these cases are:

1. The window of time between parameter substitution and command invocation.
   Allocations made by those parameter substitutions will be live but may be
   collected anyway since they are not visible in the process table.
2. ~~By extension, any commands that have delayed parts:~~
   ~~`sleep 10; map $(fn ...) $xs`. We can't read the memory of the bash~~
   ~~process, so we won't be able to know whether the `$(fn)` is still in the~~
   ~~live set.~~
   This is incorrect. Based on some tests I ran, `$()` expressions inside
   delayed commands are evaluated only once the delayed commands are.
   Therefore, the only cause of pathological delays would be something like
   this: `cat $(fn 'echo hi'; sleep 10)`, which would delay the visibility of
   the `$(fn)` form.

Bash-lambda does its best to work around these problems, but there may still be
edge cases. See `src/gc` for a full discussion of these issues, and please let
me know if you run into bugs in the garbage collector.
