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

#### Use `@ThreadSafe` and `@NotThreadSafe` a lot

#### Never share a module called `*-utils` between two processes or teams
