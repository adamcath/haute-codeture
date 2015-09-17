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
high-style guide for code. I hope you'll forgive me the pretension in the name
of a little fun.

Like all style guides, the points are not meant to be irrefutable; sometimes
they resolve arbitary decisions arbitrarily. But when taken together, they yield
a codebase with some nice properties. The examples are in Java but the rules
apply to many languages (especially those with static type-checking).




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

#### Exceptions

This is most relevant in languages like Java, where we pass strongly-typed
references around constantly and don't think much about pointers. The risk of
NPEs is high in these languages because passing typed references works great 99%
of the time. In languages with weaker type systems or manual memory management,
we must always consult the docs or the code to understand the semantics of
passed references, so we end up being more paranoid.

---

### Aside: give State its proper respect

The next several rules pertain to the careful management of state. By state, I
mean the stuff that changes over the lifetime of the program. State is what
gives computers a time dimension; it's the difference between your code, sitting
intert on the page, and your running program, crunching numbers and writing to
databases.

State is also a tremendous source of bugs in programming, because it's so hard
to reason about when looking at that inert page. Questions like "can this be
null?" or "is subsystem X initialized by the time we get here?" or "is this
class thread-safe?" are all questions about what states can be reached at what
times. So it behooves us to treat state with care; rather than smattering our
program with ever-changing values that interact, we should define abstractions
that encapsulate state changes behind an easy-to-reason-about interface.

---

### Try to make every instance field `final`

No:
```java
public class Haberdashery {
    private DbConn conn;
    private ShipmentManager shipmentMgr;
    private int maxPendingOrders;
    private List<Order> pendingOrders;
    private Order lastOrder;

    public Haberdashery(...) {
        // initialize some stuff but maybe defer others 
    }
}
```

Yes:
```java
private class Haberdashery {
    // Configuration
    private final DbConn conn;
    private final ShipmentManager shipmentMgr;
    private final int maxPendingOrders;

    // State
    private final List<Order> pendingOrders;
    private Order lastOrder;

    public Haberdashery(...) {
        // initialize everything
    }
```

This change dramatically reduces the number of states your class can be in. It
renders moot a whole class of questions like:

- Is it possible to have a `DbConn` but no `ShipmentManager`? You only need to
  look in the constructor to see how they're initialized (and if you're
  following "never return `null`", it's even more obvious that you always have
  both of these). 
- Is it possible for one thread to change `maxPendingOrders` while another is
  about to throw `TooManyOrdersException`? Nope, `maxPendingOrders` it never
  changes.

In general, you'll find that readers just don't have to worry too much about an
all-final configuration section. Those are your dependencies, which are always
configured during construction, and that's really all there is to say about
them.

The real action is in the state section. Now the reader can focus on questions
that matter, like:

- Is access to `pendingOrders` synchronized in some way?
- Is `lastOrder == pendingOrders.get(pendingOrders.size() - 1)`? How do we
  enforce that? And why do we store both anyway?

Note that we made `pendingOrders` final even though it's mutable state. Why? So
nobody has to wonder whether, in addition to mutating the list, we also change
the reference to a new list. Answer: the compiler won't let me.

Implementing this change is easy. Any time you don't plan on changing a field,
declare it final. Consider whether it's part of the mutable state of the class
or the immutable configuration and put it in the appropriate section.

#### Exceptions

Part of your program is actually the mutable state. That part can't be final
(although you can often use a final reference to a mutable object).

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
can share `Shirt` instances, but it's obvious (since all the fields are final)
that nobody will see each other's changes unless you explicitly share a new
instance. You can make all the `Shirt`s you like, but the DB will see it iff you
call a `persist()` method.

The second example above demonstrates how to deal with relationships.

By the way, making all these Builder classes in Java is a real pain. If anyone
has ideas on how make that easier, I'd love to hear them.

---

### Don't use the `Singleton.getInstance()` pattern

No:
```java
public class Haberdashery {
    public Order createOrder(Customer c) {
        DbConn.getInstance().query("INSERT INTO orders...");
        ShipmentManager.getInstance().createShipment(c.address, ...);
    }
}
```

Yes:
```java
public class Haberdashery {
    // Configuration
    private final DbConn conn;
    private final ShipmentManager shipmentMgr;

    @Inject
    public Haberdashery(DbConn conn, ShipmentManager shipmentMgr) {
        this.conn = conn;
        this.shipmentMgr = shipmentMgr;
    }

    public Order createOrder(Customer c) {
        conn.query("INSERT INTO orders...");
        shipmentManager.createShipment(c.address, ...);
    }
}

@Singleton
public class ShipmentManager {
    ...
}
```

Yes:
```java
public class Haberdashery {
    public Order createOrder(Customer c,
                             DbConn conn, 
                             ShipmentManager shipmentMgr) { 
        ... 
    }
}
```

#### Rationale

The static singleton pattern, which is ubiquitous in Java, has significant
practical drawbacks:

1. The fact that these classes are singletons is encoded in every single client
   call. If you ever needed to change that, you're going to have to update a lot
   of code. I once worked on an editor where the document model was a singleton.
   Then we added tabs. Good times were not had.
2. It's very difficult to write a unit test for `Haberdashery` without
   implicitly testing `DbConn` and `ShipmentManager`, since there's no place the
   test can insert mock versions of those objects. See "write unit tests"
   (TODO).
3. Careful management of dependencies is our best tool for creating high-level
   structure in large systems, and this pattern makes it impossible for a reader
   to enumerate everything that this class depends on without reading every line
   of code. 
4. It permits some annoying questions: when is the earliest time I can safely
   call `getInstance()`? Will it return the same thing every time? Is it
   thread-safe? What if there are multiple classloaders in play? I have seen
   bugs created by all of these potholes. We'd like to make those questions
   irrelevant.

#### Explicit dependency management

The preferred solution is to use dependency injection to pass each class's
dependencies into its constructor. If you don't know about dependency injection
yet, just watch [this video](TODO) from Google. The "yes" solution above solves
all these problems:

1. The fact that `ShipmentManager` is a singleton is encoded in one place: its
   own class declaration. If we needed multiple `ShipmentManager`s, we wouldn't
   have to touch `Haberdashery`.
2. To unit test `Haberdashery`, we can create mock versions of its dependencies
   (using a mock framework like EasyMock or by using an interface and writing an
   alternative implementation).
3. `Haberdashery`'s dependencies are clearly stated in its constructor, so we
   can tell at a glance how it fits in to our overall system. Classes with too
   many dependencies can be spotted trivially (even through automated static
   analysis).
4. `getInstance()` is replaced with an argument passed to the constructor once,
   so none of the open questions apply.

In fairness, dependency injection does have some downsides. Instantiation of
most of the long-lived objects in your program gets done by the framework.
Usually this is extremely convenient, but it is magical and can be a hard to
debug when something goes wrong (missing a `@Singleton` annotation? Your class
may get instantiated many times!). And you are giving up tight control over
instantiation order. Again, usually that is a welcome relief from a tedious and
error-prone task, but some may find it uncomfortable.

#### Exceptions

Sometimes the codebase you're working in doesn't use dependency injection and
you think it's too disruptive to add it, or the dependencies aren't available at
startup-time (`DbConn` may very well be like this), or if you just don't like
dependency injection. In those cases, you can pass the dependencies into the
method that needs them. 

---

### In fact, never store state in static fields

No:
```java
public class Shipment {
    /** Cache of currently outstanding shipments */
    private static final Map<String, Shipment> shipmentsById = new HashMap<>();

    public static void register(Shipment s) {
        shipmentsById.put(s.id, s);
    }
}

public class Haberdashery {
    public Order createOrder(Customer c) {
        ...
        Shipment s = shipmentStore.createShipment(c.address, ...);
        Shipment.register(s);
        ...
    }
}
```

Yes:
```java
/**
 * Contains all outstanding shipments. This is used to optimize the delivery 
 * subsystem.
 */
public class ShipmentCache {
    private final Map<String, Shipment> shipmentsById = new HashMap<>();

    public void register(Shipment s) {
        shipmentsById.put(s.id, s);
    }
}

public class Haberdashery {
    private final ShipmentCache shipmentCache;

    public Haberdashery(ShipmentCache shipmentCache, ...) {
        this.shipmentCache = shipmentCache;
        ...
    }

    public Order createOrder(Customer c) {
        Shipment s = shipmentStore.createShipment(c.address, ...);
        shipmentCache.register(s);
        ...
    }
}
```

#### Rationale

Go read the rule "don't use the `Singleton.getInstance()` pattern", and then
come back and try to spot the problem with static fields. Static fields are like
fields of an implicit singleton: the class object!
`Singleton.getInstance().frobnicate()` is the same as `Singleton.frobnicate()`,
and it has the exact same drawbacks around flexibility, unit-testability,
discoverability, and semantic ambiguity.

#### Model everything

Static fields usually indicate that there is some object in the program that
hasn't been modeled yet. In my example, `Shipment.shipmentsById` was actually an
in-memory cache of shipments that are also represented in the database. If we
need such a thing, we should admit it to ourselves by creating an
`ShipmentCache` and passing it around. This solves the practical problems
(testability, etc) and also makes the program more understandable because we've
modeled each distinct entity as an object. The ideal of object-oriented
programming is a set of objects interacting with each other; this fix gets us
one step closer.

#### Exceptions

Note that I'm not talking about constants (`Math.PI`) or static functions
(`Math.max()`). Those are fine because they're sort of timeless; `max()` always
means the same thing no matter what. No need to have an instance of something to
compute `max()`.




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
thermal exhaust port is too small to fire a proton (TODO sp?) torpedo into. So
you whip out a Death Sharpie and scribble a quick `TODO` on the exhaust port so
you don't forget to come back to it.

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
always wonder if I'm supposed to keep it "working" when I'm changing things near
it.

Or are you trying to tell me "this is *not* how it works"? Thanks, guy!

Dead (unused) code is just as bad. What are you saying? "This is how it might
work some day"? Sounds like a `TODO` (see "never check in TODOs"). File a
ticket.

There is one marginally acceptable usage: "uncomment this to run against a local
database". But use it judiciously, and don't expect anyone to keep it "working"
for you. You may need to make it a supported, configurable behavior.
