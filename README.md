# Coffeescript Gotchas

These are some of the things that tripped me up when I started to write
Coffeescript.  They may be useful to new Coffeescript developers.


## Coffeescript's `for..in` is *not* Javascript's `for..in`!

Don't use Coffeescript's `for .. in` for Objects, even though you would have
done that in Javascript.

- Use Coffeescript's `for .. in` for **Arrays**.
- Use Coffeescript's `for .. of` for **Objects**.

```
for i,o in ['a', 'b', 'c']
  # i=0 o='a'
  # i=1 o='b'
  # i=2 o='c'

for k,v of {a: 10, b: 20, c: 30}
  # k='a' v=10
  # k='b' v=20
  # k='c' v=30
```

You could use this mnemonic to remember:  An item lives ***in*** a list.  Keys
are properties ***of*** objects.

Or another way to remember is that it's the exact ***opposite*** way around to
Javascript's traditional `for...in` and ES6's `for...of`!


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
results for that loop, then put `return` on the last line.

    consumePies: (pies) ->
      for p in pies
        p.consume()
      return

Alternatively, put `@` on the last line to return `this`, which will present
a [fluent interface](https://en.wikipedia.org/wiki/Fluent_interface) to the
caller.


## Using `return` to return an object literal at the end of a function

If you want to return an object with properties, this is NOT the right way:

    # BROKEN
    
    getPosition = ->
      someProcessing()
      return
        x: 10
        y: 20

This will simply return `undefined`, not the object!
It processes `return` as a single statement.
(Argh, it's like semicolon injection all over again!)

You can use `return \` but that is ugly and easy to forget.

    getPosition = ->
      someProcessing()
      return \              # Works but just NO!
        x: 10
        y: 20

I recommend you get into the habit of placing the object literal directly at
the end, without a return:

    getPosition = ->
      someProcessing()

      x: 10
      y: 20

Yes that is how Coffeescript does it!  The empty line is optional but I find it
clearer.


## You cannot shadow outer variables (within a file)

If, inside a function, you assign a variable which was declared somewhere above
the function, no local variable will be created, but the outer variable will be
assigned.

This can trip you up if you aren't careful, and has been the subject of [some](http://stackoverflow.com/questions/15223430/why-is-coffeescript-of-the-opinion-that-shadowing-is-a-bad-idea#15228602)
[debate](https://github.com/jashkenas/coffeescript/issues/2697).

How to avoid issues?  When you introduce a new variable, search to see if it
has been used anywhere else in the file (specifically in a parent or child
scope of the current scope).  If it is already used, then choose a unique
name for your new variable (or rename the existing one).


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

Since parameter parsing is greedy, not [BODMAS](https://en.wikipedia.org/wiki/Order_of_operations):

    Math.sin t/77 + Math.cos t/99         # CS, looks good

becomes:

    Math.sin(t / 77 + Math.cos(t / 99))   // JS, oh it wasn't good!

I find using brackets is clearer and avoids any ambiguity.


## You will need brackets sooner or later anyway

When you have a complex expression, you will need brackets anyway.  So your
choice comes down to:

    Math.sin(t/77) + Math.cos(t/99)       # Looks like Javascript

    (Math.sin t/77) + (Math.cos t/99)     # Looks like Lisp

Perhaps you like your code to look like Lisp, in which case, go ahead!  For
me, since CS lives in the JS world, I like to keep the expressions looking
the same.


## When to skip brackets

There is one case when skipping the parentheses is fantastic, and that is for
a "trailing callback" (when you pass a callback function as the last argument):

    fs.readFile filename, (err,contents) ->
      console.log "Contents of #{filename} are:", contents

In this case if you had used brackets for the `readFile` call, you would need
an extra line at the end with just the closing `)`.  Heinous!

    fs.readFile(filename, (err,contents) ->
      console.log "Contents of #{filename} are:", contents
    )

## Plus means something special if unevenly padded

    a + ',' + b      =>      a + ',' + b;

    a+','+b          =>      a + ',' + b;

    a +','+ b        =>      a(+',' + b);   // oops!


## You are forced to declare your callbacks in reverse

Coffeescript has no way to *declare* functions.  You must *assign* them like
any other variable.  Therefore they cannot undergo the *hoisting* that you may
be familiar with from Javascript.

This will work:

    setTimeout( -> callMeLater() , 2000 )

    callMeLater = -> alert("Done")

But this will not:

    setTimeout(callMeLater, 2000)

    callMeLater = -> alert("Done")

Why not?  Because we try to access `callMeLater` before it was assigned!

This would have actually worked fine in Javascript, because declared functions
get [*hoisted*](https://en.wikipedia.org/wiki/JavaScript_syntax#hoisting) to the
top of their scope.

Unfortunately in the case above, there won't even be an error: `setTimeout` will
be called with `undefined` and just do nothing - fiddly to debug!

So you need to define the callback first (even though it happens last):

    callMeLater = -> alert("Done")

    setTimeout(callMeLater, 2000)

In other words, you must write your code backwards!  This messes up the
**flow** when reading (and writing) the code.

One way to avoid this: don't name your callbacks unless you have to, just
define them inline.

    setTimeout( -> alert("Done") , 2000 )

Another way is to follow the first example above, and make an extra anonymous
callback function that calls your named function.

    setTimeout(callMeLater, 2000)           # The problem version

    setTimeout( -> callMeLater() , 2000 )   # The safe version (less efficient)

Another option is to defer initialisation:

    init = ->
      setTimeout(callMeLater, 2000)

    callMeLater = -> alert("Done")

    init()

Which is ok provided you: don't put things in the `init` function which the
callback will need to access later, and if using the pattern a lot, be
careful that the `init = ->` of an inner scope does not *overwrite* the
`init` of an outer scope *before* the outer `init` has been called!


## There is no ternary

In Javascript the ternary conditional expression is:

    condition ? result_if_true : result_if_false

But in Coffeescript you must use the *longer*:

    if condition then result_if_true else result_if_false

However Coffeescript does have some other shortcuts which are useful in
specific situations (e.g. the existence operator `?`).

(LiveScript introduces a nice Haskell-like `| cond = result` syntax.)


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
that assists when learning Coffeescript.  Each time you save the file, it shows
**what new changes** were made to the compiled JS file, as an diff in the
preview window.  It can help when experimenting with Coffeescript, to glance
and ensure the output is what you expected it to be.

One other small gotcha is **accidentally editing** the compiled JS file instead
of the source CS file.  (I occasionally did that, before sourcemaps, when I
opened the JS file to inspect a line number reported in a runtime error.)  To
avoid making that mistake, I have
[another little vim script](https://github.com/joeytwiddle/rc_files/blob/master/.vim/ftplugin/javascript.vim)
which makes the JS file un-editable if the "Generated by Coffeescript" message
is detected at the top.


# Related Reading

- [liaody's CS gotchas](https://gist.github.com/alanhogan/8b2f67f4bae9646e88e3) - raises some serious issues
- [CoffeeScript: less typing, bad readability](https://news.ycombinator.com/item?id=4533737) - more valid criticism
