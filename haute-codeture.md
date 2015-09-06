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

The strength of style guides is that they use simple rules that are easy to
follow. This is a style guide that operates on a higher level of abstraction; a
high-style guide for code.

The examples are in Java but the rules apply to many languages (especially those 
with static type-checking).




## Rules for preventing bugs

### Never take or return `null`. Use `Optional` instead.

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

##### Rationale

There's this loophole in Java, and most mainstream statically typed languages.
If you ask for a RayGun, you might get a RayGun&hellip;or you might get nothing.
If you say you'll return a Ponycorn to your caller, you have to return a
Ponycorn&hellip;or you can just go ahead and return a special value that *looks*
like a Ponycorn, but, if your caller tries to use it, will crash her program.
What the hell! It's like your boss saying "I'll give you your paycheck on
Wednesday" and then muttering "or I might punch you in the face instead". What's
the point of static typechecking if anything can either be what it claims to be,
or a bomb that crashes your program? 

I'm referring, of course, to `null`: the "billion dollar mistake". Seriously,
the guy who introduced `null` to object-oriented languages [calls it that]
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
explicit about the possible absence of a value, but Guava's `Optional` is a
surprisingly good comprimise. 

If you may return a `RayGun` or maybe not, declare your return type as
`Optional<RayGun>`. Your callers won't have to wonder whether you're giving them
a bomb: you've explicitly told them that you may give them a `RayGun` or maybe
nothing.

Never take `null`, either. If you have an optional argument, then declare it as
`Optional<Quadcopter>`. See, it even reads nice! (You can also overload the
function with one that doesn't take that argument). 

Of course, nothing stops you from passing `null` where `Optional<Quadcopter>` is
declared. But only an asshole would do that, and most of our colleagues aren't
assholes (if they are, deal with that first). And once you develop your `null`
allergy, you'll quickly spot this in code reviews.

You may be skeptical. It might seem like boilerplate. You might worry that
chaining together a bunch of optional method calls would be annoying. I too was
skeptical, but after seeing a big codebase run in production with for 2 years of
high volume with a single-digit number of NPE's, I'm convinced. The real *feng
shui* happens when your whole codebase eschews `null`. Only then will your
beaten-dog reflex to null-check everything start to fade. But even if you only
use `Optional` in one function, you've made that function better. 

![One year of NullPointerExceptions in production](/year-of-npes.png?raw=true)
*One year of `NullPointerException`s in our production app. How does your graph
look?*

---

### Use immutable value objects

No:
```
Shirt shirt = shirtStore.loadShirt("adam");
shirt.setColor("red");
shirt.save();
```

Yes:
```
Shirt shirt = shirtStore.loadShirt();
Shirt.Builder builder = shirt.toBuilder();
Shirt newShirt = builder.withColor("red").build();
shirtStore.updateShirt("adam", newShirt);
```

No:
```
Wardrobe wardrobe = wardrobeStore.loadWardrobe();
wardrobe.addShirt(shirt);
wardrobe.save();
```

Yes:
```
Wardrobe wardrobe = wardrobeStore.loadWardrobe();
newWardrobe = wardrobeStore.addShirt(wardrobe, shirt);
```

This one is more of a design pattern than a hard rule.

##### Rationale

Much has been said about the benefits of immutability in programming. I
particularly like Rich Hickey's (creator of Clojure) talk [The Value of Values]
(https://www.youtube.com/watch?v=-6BsiVyC1kM). I'll focus on a few practical
benefits of this pattern in Java.

Ideally, when we read code, it leaves no open questions. The semantics are
completely clear from the code. `1 + 2` is pretty unambiguous in Java. As things
get more complex, we have to work harder to make them obvious. `setColor` is so
common that most of us are numb to the ambiguity, but if you think about it, it
leaves many important questions open:

- If I call it, will it update the database right now? Do I call something else
  to persist it?
- If somebody else stashed a reference to it, are they gonna see my change?
- Is `Shirt` thread-safe? When will other threads see my mutations?
- What if these other actors act on the new version before I've persisted it?
  And what if the save call fails? Do I need to think about rolling them back?

A different approach would be to say: you can't mutate a `Shirt` in memory (use
final). If you want to change it, you make yourself a new `Shirt`, and then
persist it. The compiler now restricts you to semantically obvious usages. You
can share `Shirt` instances, but nobody will see each other's changes unless you
explicitly share a new instance. You can make all the `Shirt`s you like, but the
DB will see it iff you call a `persist()` method.

The second example above demonstrates how to deal with relationships.

By the way, making all these Builder classes in Java is a real pain. If anyone
has ideas on how make that easier, I'd love to hear them.




## Rules for effective collaboration

### Never check in `TODO`s

No:
```java
// TODO: this may be a security hole.
```

Yes:
```java
// This may be a security hole. Bug DEATHSTAR-4231 tracks this.
```
... along with a bug in the bugbase.

##### Rationale

Sometimes we don't want to do every conceivable thing right now. Sometimes we're
cutting corners to hit a deadline; sometimes you're just pretty sure this
thermal exhaust port is too small to fire a proton torpedo into. So you whip out
a Light Sharpie and scribble a quick `TODO` on the exhaust port so you don't
forget to come back to it.

But maybe you didn't come back to it, and maybe there was a small security
breach. You get hauled before the emperor. "Why didn't you tell anyone about
this?" he demands. You stammer something about writing a `TODO`, but then
getting interrupted to repair a trash compactor. Lightening begins to spark from
the emperor's hands and you feel a mysterious tightening around your
neck&hellip;

##### Out of sight, out of mind

How can you remember to do something when your only reminder is buried in
thousands of lines of code?

The solution is simple. File a ticket. Then you'll prioritize the ticket during
your normal planning processes. Reference it in the code so surprised readers
know it's tracked. If the ticket would just be too embarassing ("fix glaring
security hole"), you should probably just fix it now.

---

### Never check in commented-out or unused code

No:
```
// stetsonHat.doth()
```

Yes:
```
```

Code is a message from its writers to its future maintainers. What message does
commented-out code send? The most common is "this is how it used to work". Well,
that's what source control history is for. Really. If I want to know how it used
to work, I will look in `git log`. The commented-out code is clutter, and I
always wonder if I'm supposed to keep it working when I'm changing things near
it.

Or are you trying to tell me "this is *not* how it works". Thanks, guy!

Dead (unused) code is just as bad. What are you saying? "This is how it might
work some day"? Sounds like a `TODO`. File a ticket.

There is one marginally acceptable usage: "uncomment this to run against a local
database". But use it judiciously, and don't expect anyone to keep it "working"
for you. You may need to make it a supported, configurable behavior.
