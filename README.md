# Coffeescript Gotchas

These are some of the things that tripped me up when I started to write
Coffeescript.  They may be useful to new Coffeescript developers.


* auto-gen TOC:
{:toc}


## Coffeescript's `for..in` is *not* Javascript's `for..in`!

Don't use Coffeescript's `for .. in` for Objects, even though you would have
done that in Javascript.

- Use Coffeescript's `for .. in` for **Arrays**.
- Use Coffeescript's `for .. of` for **Objects**.

```
for i,o in [ 'a', 'b', 'c' ]
  # i=0 o='a'
  # i=1 o='b'
  # i=2 o='c'

for k,v of { a: 10, b: 20, c: 30 }
  # k='a' v=10
  # k='b' v=20
  # k='c' v=30
```

It's not too hard to remember.  An item lives ***in*** a list.  Keys are properties
***of*** objects.


## Accidentally returning a list comprehension

If the last statement in your function is a loop, in many cases Coffeescript
will believe that you wanted to return the result of that loop (even though the
loop is not assigned to any variable).

    consumePies: (pies) ->
      for p in pies
        p.consume()

That generates and returns a new array of the things `p.consume()` returned.
There is memory and processing overhead to doing this, but it probably won't
break your program, so you won't notice Coffeescript is doing it unless you
check the code.

If you actually wanted to return nothing, and don't want to build an array of
results for that loop, then put `return` or `undefined` on the last line.

    consumePies: (pies) ->
      for p in pies
        p.consume()
      undefined

More recent versions of Coffeescript have become better at avoiding this,
although I don't know how the decision is made.


## Using `return` to return an object literal at the end of a function

This can cause you more trouble than it's worth.

    myFunc = ->
      someProcessing()
      return
        a: 10
        b: 20

In earlier versions of Coffeescript, this will simply return `undefined`, not
the object!  It processes `return` as a single statement.  (It's like semicolon
injection all over again!)

You can use `return \` but that is ugly and easy to forget.

    myFunc = ->
      someProcessing()
      return \              # Works but just NO!
        a: 10
        b: 20

I recommend you get into the habit of placing the object literal directly at
the end without a return:

    myFunc = ->
      someProcessing()

      a: 10
      b: 20

Yes that is how Coffeescript does it!  The empty line is optional but I find it
clearer.


## You cannot shadow outer variables (within a file)

If, inside a function, you assign a variable which was declared somewhere above
the function, no local variable will be created, but the outer variable will be
assigned.

This can trip you up if you aren't careful, and has been the subject of some
debate.

How to avoid issues?  When you introduce a new variable, consider searching to
see if it has been used anywhere else in the file (specifically in a parent or
child scope of the current scope).


## Arguments may be passed without brackets - is that good?

For example you can do `parseInt '5', 10` but a call without arguments *must
always* use `()`s, e.g. `Math.random()`.  Sometimes I would forget, and type
something like:

    print "Hi mum"
    print                        # Wrong - does not call print()!
    print "Y U no blank line?"

or even dumber:

    x = Math.cos angle
    y = Math.sin angle
    z = Math.random              # Wrong - z is now a function not a number!

The solution is to use `print()` and `Math.random()` for the no-argument calls.
This looks inconsistent.  Partly for that reason, I use parentheses on most of
my function calls.

    print("Hi mum")
    print()                      # Now works fine
    print("Y U no blank line?")

    x = Math.cos(angle)
    y = Math.sin(angle)
    z = Math.random()            # and I didn't have to think about it!


## Coffeescript is greedy when parsing parameters

Another reason to use brackets.  Without brackets, there is ambiguity regarding
where your parameters are going, requiring extra cognition for the developer.
Sometimes arguments you intended for one function will actually be passed to
the other.  For example:

    subtractNumbers 12, addNumbers 5,7         # Does that mean

    subtractNumbers(12, addNumbers(5,7))       # this

    subtractNumbers(12, addNumbers(5), 7)      # or this?

Since parameter parsing is greedy, not BODMAS:

    Math.sin t/77 + Math.cos t/99         # CS, looks good

becomes:

    Math.sin(t / 77 + Math.cos(t / 99))   // JS, oh it wasn't good!

I find using brackets is clearer and avoids any ambiguity.


## You will need brackets sooner or later anyway

When you have a complex expression, you will need brackets anyway.  So your
choice comes down to:

    Math.sin(t/77) + Math.cos(t/99)       # Looks like Javascript

    (Math.sin t/77) + (Math.cos t/99)     # Looks like Lisp

Perhaps you like your code to look like Lisp, in which case, go ahead!  But
since CS lives in the JS world, I like to keep the expressions looking the
same.


## When to skip brackets

There is one case when skipping the parentheses is fantastic, and that is for
a "trailing callback" (when you pass a callback function as the last argument):

    fs.readFile filename, (err,contents) ->
      console.log "Contents of #{filename} are:", contents

In this case if you had used brackets for the `readFile` call, you would need
an extra line at the end with just the closing `)`.  Yuck!


## Plus means something special if unevenly padded

    a + "," + b      =>      a + "," + b;

    a+","+b          =>      a + "," + b;

    a +","+ b        =>      a(+"," + b);   // oops!


## You are forced to declare your callbacks in reverse

Coffeescript has no way to *declare* functions.  You must *assign* them like
any other variable.  Therefore they cannot undergo the *hoisting* that you may
be familiar with from Javascript.

This will work:

    setTimeout( -> callMeLater() , 2000 )

    callMeLater = -> alert("Done")

But this will not:

    setTimeout( callMeLater , 2000 )

    callMeLater = -> alert("Done")

Because we try to access `callMeLater` before it was assigned!

This would have actually worked fine in Javascript.  (Although that is not
always advisable either: it is
[ambiguous](https://github.com/joeytwiddle/code/blob/master/other/javascript/snippets/declare_callback_after_passing_it.html)
inside conditional blocks.)

Unfortunately in the case above, there won't even be an error, just a big black
nothing - fiddly to debug!

So you need to define the callback first (even though it happens last):

    callMeLater = -> alert("Done")

    setTimeout( callMeLater , 2000 )

In other words, you must write your code backwards!  This messes up the
**flow** when reading (and writing) the code.

One way to avoid this: don't name your callbacks unless you have to, just
define them inline.

    setTimeout( -> alert("Done") , 2000 )

Another way is to follow the first example above, and make an extra anonymous
callback function that calls your named function.

    setTimeout( callMeLater , 2000 )        # The problem version

    setTimeout( -> callMeLater() , 2000 )   # Safe but less efficient

Another option is to defer initialisation:

    init = ->
      setTimeout( callMeLater, 2000 )

    callMeLater = -> alert("Done")

    init()

Which is fine provided you don't want to make things available to the later
callback which were in the scope of the `init` function.  Warning: if you use
this pattern in parent and child functions, you may find yourself sharing the
`init` var between scopes (which is often not a problem, but will break things
if an inner `init` is defined before the outer `init` is called).


## There is no ternary

In Javascript the ternary conditional expression is:

    condition ? result_if_true : result_if_false

But in Coffeescript you must use the *longer*:

    if condition then result_if_true else result_if_false

However Coffeescript does have some other shortcuts which are useful in
specific situations (e.g. the existence operator `?`).

You *could* write a function to do ternary but that would be silly:

    ternPre  =   (cond,a,b) -> if cond then a else b
    ternPost = (cond,fA,fB) -> if cond then fA() else fB()

    x = if x<5 then x else 10-x

    x = ternPre x<5, x, 10-x             # Always evaluates both results

    x = ternPost x<5, (->x), ->10-x      # Longer than if-then-else!

LiveScript introduces a nice case-like `| cond = result` syntax.


# Tips (not gotchas)

## Immediately calling an anonymous function

If you want to call a function immediately after defining it:

    myObject = ( ->
      privateThing = 3
      publicThing: -> privateThing*2
    )()

You can avoid the brackets by using the `do` keyword:

    myObject = do ->
      privateThing = 3
      publicThing: -> privateThing*2


# Assistance for Vim users

I wrote a
[helpful Vim plugin](https://github.com/joeytwiddle/rc_files/blob/master/.vim/ftplugin/coffee.vim)
that assists when writing Coffeescript.  Each time you save the file, it shows
what **changes** were made to the compiled JS file as a diff.  I hope it can
help!

One other small gotcha is **accidentally editing** the compiled JS file instead
of the source CS file.  (I occasionally do that when I have opened the JS file
to inspect a line number reported in a runtime error.)  To avoid making that
mistake, I have
[another little vim script](https://github.com/joeytwiddle/rc_files/blob/master/.vim/ftplugin/javascript.vim)
which makes the JS file un-editable if the "Generated by Coffeescript" message
is detected at the top.

