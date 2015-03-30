# Exploring ML-Style Modular Programming in Scala

(DRAFT)

I recently watched a [talk by Martin
Odersky](https://www.youtube.com/watch?v=P8jrvyxHodU) in which he boils
Scala down to what he considers to be the essential parts of the
language. In it he remarks that Scala is designed to be a modular
programming language and its modular abstractions are greatly inspired
by modular programming in ML (SML). I found this intriguing, because
I regard SML-style modular programming as a great way to organise
software when doing programming-in-the-large. If Prof. Odersky's
assertion about modular progamming in Scala is correct, SML-style
modules might be a great fit with Scala.

As is usually the case, I went searching for what others have had to say
on the matter. I found a great
[post](http://io.pellucid.com/blog/scalas-modular-roots) by @dwhjames
showing several attempts and approaches at encoding ML-style modules in
Scala, and tries out several syntactic 'styles' of Scala to see which
one might 'feel' better in use. James starts with Odersky's central
premise, which I interpret as the following key points:

  - Scala object = ML module

  - Scala trait = ML signature

  - Scala class = ML functor

He then goes on to explore actually implementing modules using the
driving example of a `set` data structure from Okasaki's Purely
Functional Data Structures (appropriate, as Okasaki famously uses SML in
the code examples for his data structures).

I also found a very thoughtful [answer and
discussion](http://stackoverflow.com/q/23006951/20371) to the same
question, posed on Stack Overflow. The answer and discussion here are
just a gold mine of information about Scala's theoretical issues with
handling ML-style modules. The key points I took away from them:

  - ML functor = Scala function. This is slightly more elegant than the
    cumbersome declaration of a new class; but it does require a little
    finesse to avoid typechecker traps.

  - Scala's type relationships are _nominal_ as opposed to ML's
    _structural_ relationships. In practice this means we will need to
    declare and define all our types and traits before trying to set up
    relationships between them using functors. This is admittedly a
    limitation, but perhaps one worth living with given the code
    organisation and maintainability benefits we gain in exchange.

Finally, I found a [fantastic talk](https://youtu.be/oJOYVDwSE3Q) by
@chrisamaphone on ML modules, their use, and the type theory behind
them. I'm still mulling all the new ideas over, but so far I've managed
to take away the following main points:

  - SML functors are _generative_ functors, so called because they
    _generate_ new modules as return values for each time they're
    called. This is true even if they're called with the exact same
    arguments each time. Similar to Haskell's `newtype A = Int` and
    `newtype B = Int` creating two completely distinct types `A` and `B`
    despite being literally the same type.

  - SML functors are actually modelled using formal logic techniques
    which I won't pretend to understand--just appreciate.

  - It's possible and sometimes even desirable to get different results
    from two modules which have been created by the same functor. This
    despite the general idea behind functors being to abstract
    implementation away from interface, implying that all
    implementations should return the same results (otherwise the
    interface 'breaks its promise').

## A Basic Module

As a sort of review, let's look at a simple SML module and beside it a
Scala port. This example is adapted from a fantastic paper on the
[Essentials of Standard ML
Modules](http://www.itu.dk/courses/FDP/E2004/Tofte-1996-Essentials_of_SML_Modules.pdf)
(PDF), which explains SML modules from the ground up. It implements a
_finite map_ from integers to values of some arbitrary type. In other
words, a vector. On a side note, the interesting thing about this data
structure is that it's implemented purely using function
composition.

```scala
/* structure IntFn =              */ object IntFn {
/*   struct                       */   case class NotFound() extends Exception()
/*     exception NotFound         */   type T[A] = Int => A
/*     type 'a t = int -> 'a      */
/*                                */   def empty[A]: T[A] =
/*     fun empty i =              */     (i: Int) => throw NotFound()
/*       raise NotFound           */
/*                                */   def get[A](i: Int)(x: T[A]) = x(i)
/*     fun get i x = x i          */
/*                                */   def insert[A](k: Int, v: A)(x: T[A]) =
/*     fun insert (k, v) x i =    */     (i: Int) => if (i == k) v else get(i)(x)
/*       if i = k then v else x i */ }
/*   end;                         */
```

With this implementation, we can do things like:

    scala> (IntFn.insert(1, "a") _ andThen IntFn.get(1)) { IntFn.empty }
    res7: String = a

A few points to take away from this:

  - Scala's generic methods will require type parameters, which is
    really clunky, unless you manage to define the type parameter
    elsewhere, as we will see later.

  - Scala is somewhat denser than SML for the equivalent functionality.
    To me, this is mostly a result of Scala's weaker type inference that
    forces us to specify a lot more.

  - Scala doesn't really optimise for using curried functions, so if we
    want to use that style function composition using `andThen` and
    partial application is the [most
    idiomatic](http://stackoverflow.com/a/20574722/20371) way to do it.
    In OCaml or F# we could use the 'forward-pipe' operator (`|>`). We
    can define a forward-pipe operator in Scala (and people have); but
    the implementation isn't something you want running in production.

## The Module Signature

The next step in the evolution of a module is usually to extract its
signature:

```scala
/* signature INTMAP =                       */ import scala.language.higherKinds
/*   sig                                    */
/*     exception NotFound                   */ trait IntMap[A] {
/*     type 'a t                            */   type NotFound
/*                                          */   type T[_]
/*     val empty: 'a t                      */
/*     val get: int -> 'a t -> 'a           */   val empty: T[A]
/*     val insert: int * 'a -> 'a t -> 'a t */   def get(i: Int)(x: T[A]): A
/*   end;                                   */   def insert(k: Int, v: A)(x: T[A]): T[A]
/*                                          */ }
```

Now we start making some trade-offs in Scala. Some points to take away:

  - We're able to clean out the type parameters from all the methods,
    but we're now passing in the type parameter into the trait. And,
    because the type `T` is abstract (we don't know in the trait how
    implementors will define it), we have to say that it accepts any
    unknown type as a parameter, and so we have to now enable
    higher-kinded types.

  - We express `empty` as a `val` instead of as a `def` because we want
    it to return the exact same thing each time; so no need for a
    function call to do that. We couldn't do this with the object
    version before because `val`s can't accept type parameters
    (this is a hard fact of Scala syntax). This also forces us to move
    out the type parameter to the trait, so that it's in scope by the
    time we start declaring `empty`.

After that, the next step is to express the module's implementation in
terms of the signature:

```scala
/* structure IntFn :> INTMAP =    */ trait IntFn[A] extends IntMap[A] {
/*   struct                       */   case class NotFound() extends Exception()
/*     exception NotFound         */   type T[A] = Int => A
/*     type 'a t = int -> 'a      */
/*                                */   override val empty =
/*     fun empty i =              */     (i: Int) => throw NotFound()
/*       raise NotFound           */
/*                                */   override def get(i: Int)(x: T[A]) = x(i)
/*     fun get i x = x i          */
/*                                */   override def insert(k: Int, v: A)(x: T[A]) =
/*     fun insert (k, v) x i =    */     (i: Int) =>
/*       if i = k                 */       if (i == k) v else get(i)(x)
/*         then v                 */ }
/*         else get i x           */
/*   end;                         */
```

Can you tell the crucial difference between the original SML and the
Scala translation? Hint: it's right at the first word of the first line
of each definition.

We express the SML module directly as a concrete structure, while we
express the Scala module as an abstract trait that takes a type
parameter. Remember how we mentioned earlier that in the Scala version
we needed to pass in a type parameter? I mentioned some basic reasons
earlier, but there are better reasons that that we'll explore later. For
now, I'll mention that we want Scala modules to be as flexible as
possible, and that leads to design decisions that inevitably lead us to
passing in type parameters.

Well, as a consequence of these decisions, we also aren't able to
directly create a module (i.e., a Scala object) that can work on any
given type. Scala objects can't take type parameters; they can only be
instantiated with concrete types (well, unless you use existential
types, which is advanced type hackery that I want to avoid).

So, we have to define something which _can_ take a type parameter; and
the choices are a trait or a class (if we're defining something at the
toplevel, that is). I went with a trait partly to minimise the number of
different concepts I use, and partly to emphasise the abstract nature of
this 'module template', if you will.

Now, we can instantiate _concrete_ Scala modules (objects) with a type
parameter of our choosing, and within some scope (not at the toplevel):

```scala
object MyCode {
  val IntStrFn: IntMap[String] = new IntFn[String] {}

  (IntStrFn.insert(1, "a") _ andThen IntStrFn.get(1) _ andThen print) {
    IntStrFn.empty
  }
}
```

Notice how:

  - We constrain the `IntStrFn` module to only expose the `IntMap`
    interface, just as we constrained the SML `IntFn` module to only
    expose the `INTMAP` signature using the constraint operator `:>`. As
    a quick reminder, ML calls this 'opaque signature ascription' and we
    use it to get the benefit of hiding our implementation details.

    There is another type of ascription, 'transparent' ascription, which
    means 'the module exposes _at least_ this signature, but possibly
    also more'. We get that in Scala by simply leaving out the type
    annotation from the module declaration and letting Scala infer a
    subtype of the signature trait for our module.

    TODO: point reader to further reading on this topic.

  - We define the module (`IntStrFn`) inside an object `MyCode` because
    in Scala, `val`s and `def`s can't be in the toplevel--they need to
    be contained within some scope. In practice we can easily work
    around that restriction by defining everything inside some object
    and then importing all names from that object into the toplevel.

  - The Scala implementation ends up using two traits for two levels of
    abstraction (the module interface and the implementation using a
    representation of composed functions), which is somewhat sensible.

  - We use `override` extensively in the `IntFn` trait to enable
    C++-style virtual method dispatch for when we're holding several
    different `IntMap` instances with different concrete implementations
    and we want to operate on all of them uniformly and have them just
    do the right thing automatically.

  - We implement the `IntStrFn` module as actually an anonymous class
    that extends the `IntFn` trait and passes in the concrete type as
    the type parameter. The class has an empty body because it extends a
    trait which defines all its methods and values already.

## A Detour into Module Opaque Types

If you evaluate all the Scala traits and modules shown upto this point
in the REPL, and then evaluate the following code:

```scala
import MyCode._
IntStrFn.empty
```

You'll get back something like:

    res: MyCode.IntStrFn.T[String] = <function1>

This is one of the elegant things about ML-style modules. Each module
contains all definitions and _types_ it needs to operate, in a single
bundle. Traditionally, the module's primary type is just called `T`, so
`IntStrFn.T` has the sense that it's the `IntStFn` module's primary
type.

This type alias that you get from the module, known as an opaque type,
doesn't give you any information or operations on itself. It limits you
to using values of the type in only _exactly_ the operations that the
module itself provides. And that's a _great_ thing for modularity and
information hiding.

You might point out that the REPL actually tells you that the type is
really a `Function1`, so you immediately know you can call it and do
certain other operations on it. But that's a detail of how the REPL
prints out the values of arbitrary objects after evaluating them. It's
not something you'll have access to when you're actually building
programs to run standalone.

To see a concrete example of how ML-style opaque types are great at
helping you make compiler-enforced guarantees in your programs, see
Prof. Dan Grossman's excellent course using SML and his
explanations of [data type
abstraction](http://courses.cs.washington.edu/courses/cse341/13sp/unit4notes.pdf)
(PDF, pp. 3--6).

## Functors

Now that we've set up all the building blocks of modules, we can tackle
one of ML's most flexible methods for modular code organisation:
_functors,_ functions which build modules. To illustrate functors, I'll
re-implement a functorised module from an article I mentioned earlier,
@dwhjames'
[adaptation](http://io.pellucid.com/blog/scalas-modular-roots) of a
functional set data structure. Here I show and explain my version.

```scala
trait Ordered[A] {
  type T = A

  def compare(t1: T, t2: T): Int
}
```

This is almost exactly the same as @dwhjames' `Ordering` trait; it's
just that I've tried to stick closer to the SML names wherever possible.
In essence, it sets up a signature for a module that can define a
comparator function for any given datatype. To actually define the
comprator, you just need to create a concrete module, an example of
which we will see later.

```scala
trait MySet[A] {
  type E = A
  type T

  val empty: T
  def insert(e: E)(t: T): T
  def member(e: E)(t: T): Boolean
}
```

This is a signature for a module which can implement a set. You'll
notice that I'm using _both_ type parameters and abstract types in my
signatures so far. The reason for using a type parameter is as follows.
The set functions `insert` and `member` declare parameters of type `E`
(= Element), which is aliased to type parameter `A`. This allows
concrete modules to pass in values of the concrete type they've been
instantiated with to the functions, and these are internally 'seen' as
values of type `E` without any need for type refinemint or other type
trickery.

The reason for using the type alias `E` when we already have the type
parameter `A` is as follows. If and when we do implement a concrete
module with the `MySet` signature, e.g.: `val IntSet: MySet[Int] = new
MySet[Int] { ... }`, we'll be able to define both `insert` and `member`
using the _exact_ same types that we have in the trait. We won't have to
say e.g. `def insert(e: Int)(t: T) = ...`; we'll just say `def insert(e:
E)(t: T) = ...`. This reduces the possibility for simple copy-paste
errors and such.

Of course, in this example we're not implementing a concrete module
directly, so that doesn't matter so much. Back to the example:

```scala
object Modules {
  val IntOrdered: Ordered[Int] = new Ordered[Int] {
    override def compare(t1: T, t2: T) = t1 - t2
  }
```

Here we're forced to start putting our concrete modules inside a
containing scope because as mentioned earlier Scala `val`s can't reside
in the toplevel.

We could have declared `IntOrdered` in the toplevel using `object
IntOrdered extends Ordered[Int] { ... }` but that wouldn't have achieved
opaque signature ascription; the module wouldn't have been as tightly
controlled as it is with just the type `Ordered[Int]`. So we'll define
it inside a container module (`Modules`) and import everything from
`Modules` into the toplevel.

```scala
  def UnbalancedSet[A](O: Ordered[A]) = // 1
    new MySet[A] { // 2
      sealed trait T
      case object Leaf extends T
      case class Branch(left: T, e: E, right: T) extends T

      override val empty = Leaf

      override def insert(e: E)(t: T) =
        t match {
          case Leaf => Branch(Leaf, e, Leaf)
          case Branch(l, x, r) =>
            val comp = O.compare(e, x) // 3

            if (comp < 0) Branch(insert(e)(l), x, r)
            else if (comp > 0) Branch(l, x, insert(e)(r))
            else t
        }

      override def member(e: E)(t: T) =
        t match {
          case Leaf => false
          case Branch(l, x, r) =>
            val comp = O.compare(e, x) // 4

            if (comp < 0) member(e)(l)
            else if (comp > 0) member(e)(r)
            else true
        }
    }
```

The rest of the implementation is almost exactly the same as in
@dwhjames' article. I'll just point out the interesting bits from our
perspective, which I've marked above with the numbers:

  1. This is the start of the functor definition. Notice how it's just a
     normal Scala function which happens to take what we think of as a
     concrete module as a parameter; and

  2. It happens to return what we think of as a new concrete module. Of
     course, at the level of the language syntax, they're both just
     simple objects that implement some interface.

  3. And also 4. Here we actually use the comparator function defined in
     the `Ordered` signature to figure out if the value we were just
     given is less than, greater than, or equal to, values we already
     have in the set. These two usages are exactly why the
     `UnbalancedSet` functor has a dependency on a module with signature
     `Ordered`. And the great thing is it can be any module that does
     anything, as long as it ascribes to the `Ordered` signature.

     For those with a Haskell background, this is basically the ML
     equivalent of Haskell' typeclasses. Of course, here it's
     implemented in Scala, which means that Scala has at least two
     different ways of encoding Haskell's typeclasses (functors and
     implicits). Of course, functors are (ahem) a lot more explicit than
     implicits.

```scala
  // UIS = UnbalancedIntSet
  val UIS: MySet[Int] = UnbalancedSet(IntOrdered)
```

This is where we actually define a concrete module which behaves as a
set of integers implemented as an unbalanced tree. All the expecetd
operations work:

    scala> (UIS.insert(1) _ andThen UIS.insert(1) _ andThen UIS.insert(2) _ andThen UIS.member(1)) { UIS.empty }
    res0: Boolean = true

```scala
  // Slay the compiler:
  //val UIS = UnbalancedSet(IntOrdered)
}
```

This is something that actually kills the compiler. It looks like
something about returning an object of a new anonymous class from a
function is too much for Scala's type inference to figure out. Note that
it's solid as a rock if you annotate the type, which we did in the above
uncommented code.

