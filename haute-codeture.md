Haute Codeture
==============

A high-style guide for code (mostly Java)
-----------------------------------------

Ah, style. The topic of innumerable water-cooler conversations, religious wars,
and Linus asshole-tearings. In my corner of the industry, we seem to have gotten
surprisingly mature about the topic. People use standard style guides (like
Google's), respect the style of the codebase they're working in, and defer to
project leaders.

This is great progress, but I still find myself making the same comments in code
reviews over and over. Traditional style guides help make your code readable at
a low level of abstraction: where curly braces go, how to capitalize things, and
other lexical or syntactic details. I wanted a new kind of style guide that
helps you avoid bugs and write *semantically* readable code.

This is a style guide that operates on a higher level of abstraction. A
high-style guide for code.

The examples are in Java but the rules apply to many languages.

## The rules

#### Never take or return `null`. Use `Optional` instead.

No:
```java
public RayGun loadRayGun(int id) {
    Rowset rows = db.query("SELECT ...");
    if (rows.isEmpty()) {
        return null;
    }
    return rows.get(0);
}
```

Yes:
```java
public Optional<RayGun> loadRayGun(int id) {
    Rowset rows = db.query("SELECT ...");
    if (rows.isEmpty()) {
        return Optional.absent();
    }
    return Optional.of(rows.get(0));
}
```

##### A brief history of static typing

Back in the day, people wrote in assembly languages. Put this in a register,
shift that register onto the stack, add the two top stack frames and put it in
this register, et cetera. This basically sucked. One of the numerous problems
was that you had to keep track of what types of things were stored where.
Register 17 has an 16-bit floating point number. Register 4 has a 32-bit
two's-complement integer. The 3rd stack element contains the address of a byte
of memory which contains the first ASCII character in a null-terminated string.

That was super annoying to remember, so we invented named variables, functions,
and static types. Now you didn't have to remember where everything was and how
it was stored. Instead, there were a set of names that were in scope at any
given time, and their types were right there in the code, next to the names. Not
only that, but the compiler would check that nobody passed you a BroadSword when
you expected a RayGun. What a victory for productivity!

But there's a loophole in Java (and most mainstream statically typed languages).
If you ask for a RayGun, you might get a RayGun...or you might get nothing. If
you say you'll return a Ponycorn to your caller, you have to return a
Ponycorn...or you can just go ahead and return a special value that *looks* like
a Ponycorn, but, if your caller tries to use it, will crash her program. What
the hell, guys! It's like your boss saying "I'll give you your paycheck on
Wednesday" and then whispering "or I might punch you in the face instead".
What's the point of type checking if anything can either be what it claims to
be, or a bomb that crashes your program? 

I'm referring, of course, to `null`: the "billion dollar mistake". Seriously,
the guy who introduced `null` to object-oriented languages calls it that
(https://en.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions). To wit:

> This [(`null`)] has led to innumerable errors, vulnerabilities, and system
> crashes, which have probably caused a billion dollars of pain and damage in
> the last forty years.
>
> &mdash; *Tony Hoare*

`NullPointerException`s and segmentation faults are commonplace in the extreme.
"NPE when I cast 'Intimidating Shout'" fills our bugbases and "can this argument
be `null`?" peppers our code reviews. It's time to stop this madness.

##### `Optional` to the rescue

The best solution is to use a programming language that forces you to be
explicit about the possible absence of a value, but just using Guava's
`Optional` is a surprisingly good comprimise. 

If you may return a `RayGun` or maybe not, declare your return type as
`Optional<RayGun>`. Your callers won't have to wonder whether you're giving them
a bomb: you've explicitly told them that you may give them a `RayGun` or maybe
nothing.

Never take `null`, either. If you have an optional argument, then declare it as
`Optional<Quadcopter>`. See, it even reads nice! (You can also overload the
function with one that doesn't take that argument). 

You may be skeptical. It might seem like boilerplate, or like chaining together
a bunch of optional method calls would be annoying. I too was skeptical, but
after seeing a big codebase run in production with for 2 years of high volume
and a single-digit number of NPE's, I'm convinced. The real *feng shui* happens
when your whole codebase eschews `null`, but even if you only use it in one
function, you've made that function better. 

!(/year-of-npes.png?raw=true "One year of `NullPointerExceptions` in production")
