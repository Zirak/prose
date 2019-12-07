# A Journey Into the Land of Nothing

## Intro

XXX What will we be talking about? Against your nature, try to be succinct

There's a lot of talking about handling missing values. You've probably dealt with `null`s, `undefined`s, `nil`s, and `None`s, or whatever your favourite language calls this concept. You may have also dealt with `Maybe`s and `Optional`s (but if not, that's fine too).

As it turns out, a lot of times we're dealing with values which may just not be there. User inputs, configurations, and most importantly, business requirements. You name it. Just a chaotic mess of things which may or may not be. Quite an existential affair, this whole programming business.

In this article I wish to take you on a journey. A journey to the rickety shores of the Land of Nothing. Your map to this journey:
- The origin of `null`
- A data structure named `Maybe`
- Function input/output guarantees and breaking changes
- The concept of `null`able types
- Why you can throw them all out the window
- A deeper look into the nature of nothing, types, and functions
- Touring through possible solutions
- Appendices detailing why similar solutions don't make the cut

This article may seem too long. It kind of is. I attempt to sum up the things I've been thinking about over the past few years in dealing with `null` values and the concept of nothing. Turns out there's a lot to say and unpack here, who knew?
Also, the appendices ended up taking up a lot of space. I should get better at that.

Hang on to something as we go, onwards, into the Land of Nothing!

## null

XXX

In this section, we learn about `null`.
Keep this section short. Link to another article to get the user acquainted.

- [The Billion Dollar Mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/).
- A `null` is a value in every type, but it does not operate like any other value in the type.

## Maybe

XXX

In this section, we learn about `Maybe`s and show how they allow us to speak about `null`.
This section can be longer long, but not overly. This isn't a `Maybe` tutorial. Link to another article explaining `Maybe`s and `Optional`s better.

- More importantly, they allow us to talk about values without having to talk about `null`s. We just call the `map`, and we're in value-land without a care in the world.

## Function guarantees

Let's imagine this function signature:

```
foo(x: 1 | 2 | 3) => 'a' | 'b'
```

A function `foo` accepts a single argument, either `1`, `2`, or `3`, and returns a single value, either `'a'` or `'b'`.

Imagine this is a library function, `foo v1`, and we're about to release `foo v2`.

What would we consider a breaking change to this signature? How can this function break its callers?

By either *strengthening the input* or *loosening the output*.

If we receive less inputs (strengthen), we break:
```
// ----v
foo(x: 1) => 'a' | 'b'
```

This breaks existing calls to `foo(2)` and `foo(3)`. We *strengthen the input*, and suddenly previously valid calls are no longer.

Likewise, we can break by returning more outputs (loosen):

```
foo(x: 1 | 2 | 3) => 'a' | 'b' | 'c'
// -------------------------------^
```

Would break existing calls to `foo`, as they now have to deal with a previously unseen value.

The inverse of this is true. A non-breaking change is *loosening the input*, or *strengthening the output*:

```
foo(x: 1 | 2 | 3 | 4) => 'a' | 'b'
```

We can now accept more values. As existing callers never called us with any, they're fine.

```
foo(x: 1 | 2 | 3) => 'a'
```

Likewise, as our caller knows to deal with all of our previously returned values, it can deal with a smaller set of them.

To put it into a table:


|        | Strengthen | Weaken |
|--------|------------|--------|
| Input  | :(         | :D     |
| Output | :D         | :(     |

This is codified to us from [Postel's Law](https://en.wikipedia.org/wiki/Robustness_principle):

> Be conservative in what you send, be liberal in what you accept.

Meditate on the consequences for a minute. Think about how you saw this happen with your own two eyes, countless times before. These principles are the basis for the rest of this discussion.

## Maybe went postal

XXX heading title

`Maybe`s break the guarantees guidelines we outlined above. Consider:

```
lookupPhoneNumber(person: { addr: string }) => Maybe(number);
lookupPhoneNumber({ address: '10 Whatever Road' }).map(phone => {
    console.log('ring ring!', phone);
});
```

We receive a person with an address and look up their number. Sometimes an address has an associated phone number. Maybe it does, maybe it doesn't.

Now let's *weaken the input*, a non-breaking change, allowing us to deal with a person who doesn't have an address:

```
lookupPhoneNumber(person: { addr: Maybe(string) }) => Maybe(number)

// Error! string is not a Maybe
lookupPhoneNumber({ addr: '10 Whatever Road' }).map(phone => {
    console.log('ring ring!', phone);
});
```

*This breaks our callers*. We wanted to increase the number of allowed inputs,but in effect our existing callers broke.

Now, let's say the government came out with a law, requiring every single address to have a phone number. Since now we always return something, we try to *strengthen* our input:

```
lookupPhoneNumber(person: { addr: string }) => number

// Error! number doesn't have a `map` function
lookupPhoneNumber({ addr: '10 Whatever Road' }).map(phone => {
    console.log('ring ring!', phone);
});
```

Again, we broke our callers.

Contrast this to if we were using `null`s and not `Maybe`s, obtaining this signature:

```
lookupPhoneNumber(person: { addr: string | null }) => number | null
```

Both examples above would work perfectly well. We either add or remove the `| null`.

*Maybe is not a good enough solution: it breaks callers*.

Let's roll down our sleeves and try to tackle this `null` problem.

***Note***: Until the end of this article, I will only be talking about *inputs*, and not *outputs*. This will be explained. Do try to go through this journey with an open mind.

## Nullable Types

As it turns out, Kotlin & Swift nullable types handle `null`s perfectly.

From [Kotlin documentation on null safety](https://kotlinlang.org/docs/reference/null-safety.html):
```kotlin
var a: String = "abc"
a = null // compilation error

var b: String? = "abc"
b = null // ok

val l = b.length // error: variable 'b' can be null
val l = if (b != null) b.length else -1
// or, syntactic sugar:
val l = b?.length ?: -1

// produces an Int?
val l = b?.length
```

This fits into our guarantee principles nicely. Let's look at the phone number example above (pseudocode):

```
lookupPhoneNumber(person: { addr: string? }) => number?

const num = lookupPhoneNumber({ addr: '10 Whatever Street' });
console.log(num + 1); // compilation error
```

End of article. Let's wrap up, use Swift, and go home for the day, lounging with our favourite Cactuar plushie.

Naaah, you think I was going to let you off so easy? Look at the scrollbar, we've still got some ways to go! The downside of writing this as part of the type system is that you're using a type system.

### Type proliferation

The downside of using types is apparent if you've ever worked with a language using types. It's type proliferation and type inconsistency. The code below is paraphrased from the excellent article [Designing with types: Making illegal states unrepresentable](https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/).

Let's take our nice scapegoat person. In recent years due to the Vanlife Epidemic, physical addresses are becoming cliché, and his address could be a physical one, a virtual one (email), or he could be a luddite and have both. Imagine that. Thing is, you *always* have at least one. We change our data structure:

```
Person = {
    name: string,
    contact: Contact,
    // ...
};
Contact = {
    address?: string,
    email?: string,
};
```

We created a key `contact` which is composed of both his physical `address` and his `email`. Of course, each one (or both!) may be `null`. However, that's not helpful. In our nice little world, either the person has a address, or an email, or maybe both, but never neither.

```ts
// Doesn't care if physical or electronic
function sendBill(contact: Contact) {}
// We require the physical part
function sendPackage(contact: AddressContact) {}
// We require the electronic part
function sendEmail(contact: EmailContact) {}
// We require both physical part and electronic
function sendPackageWithConfirmation(contact: FullContact) {}
```

(The typescript people may be throwing plushies at the screen, yelling at me to use an anonymous type object. I ask of them lay down their cactuars, be patient, read through, and then read through the TS appendix.)

And what would writing the types be?

```ts
type Person = {
    name: string,
    contact: Contact,
};

type Contact = AddressContact | EmailContact | FullContact;
type AddressContact = {
    address: string,
    email?: string,
};
type EmailContact = {
    address?: string,
    email: string,
};
type FullContact = {
    address: string,
    email: string,
};
```

Yuck. Some languages make typing this easier (like F#), or using it easier (like F#, Haskell, or Rust), but none hide how we've created *four* types to represent an *or*.

Worse yet, if you've ever passed data across systems, you know how infuriating this kind of type proliferation can be. You have your notion of `EmailContact`, but the system you're talking with has *the same thing*, only it's called `OnlyEmail`, and now you have to do conversions.
We've fallen into the same trap ourselves: A system with no notion of emails, and only addresses, must go through hoops to use our address-only functions, because we've codified unrelated expectations in our types.

Some languages make the above easier (like TypeScript), or harder (like Java). Interoperability in the land of giving things names is frustrating.

I'm not dissing language creators. This is not just a hard problem, there is an inherently difficult precondition in coming up with a solution:

### Everything is a Maybe

XXX move some of this part into the conclusion. it's too ranty

We *always* get to the point where we *sometimes* don't have something. That's not the exception, that's the rule. Sometimes we don't have someone's address. Sometimes they don't have their credit card on them. If we were to faithfully represent the majority of data structures around us, we'll end up wrapping everything as a `nullable` or `Maybe` *in the base case*.

Is it fine for a person not to have an address? Maybe! Sometimes it is, sometimes it isn't. When you're ordering mail, of course not, but when you're buying a stick of gum, have at it!

Is it fine for a user to not have a user id? Maybe! If you're a database then no, but if you're logging a user in, then that's perfectly fine. You only have the user's name and password at that point, and you want to ask for the rest.

As it turns out, a lot of our functions look like this:

```ts
whatever(person: Partial<Person>) => NotSoPartial<Person>
```

We give a function a part of what we know, and it gives back some more info. We've known this for a long time. We've just rolled and unrolled this assumption over and over again in our code. Consider, how many times have you done this:

```js
var user = { name: 'Sheepy', address: 'The Barn' };
user.id = lookupUserId(user.name, user.address);
whateverThatRequiresUserID(user);
```

We're arbitrarily taking apart your nicely consolidated data structure, and taking a return value on the original one.

We're forced to break apart our notion of an object into changeable, context-specific arguments. Wouldn't it be better if we wrote this:

```ts
lookupUser(user: Partial<User>) => User
lookupUser({ name: 'Scapey', address: 'The Barn' })
// => { id: 10, name: 'Scapey', address: 'The Barn', ... }

whatever(lookupUserId(user));
```

Boom. We give in a partial user, get back a filled in user, or a *more* filled in user.

I'm going to repeat this point. Like input/output guarantees, it is a vital concept to internalise:

When creating a type, whether a property is required is undecidable. A property is `null`able based on *usage* and *context*.

This is the functional inversion again. Class-based OO languages fall short because they focus on classes, on nouns, and not on functions, on verbs. Slot-based types fall short because they focus on types, on nouns, and not on usage context, on verbs.
XXX reword

Meditate about this for a few minutes. Imagine how many functions named `loginWithEmail`, `loginWithOAuth`, `loginWith...` you've seen. How many objects you've broken apart into arguments for absolutely no reason.
Really consider this, even more than just a few minutes.
It's fine, take your time. I've been thinking about this for over a few years, and it still hasn't completely set in.

On returning from your meditations, one might wonder how this fits into our earlier discussion of strengthening and weakening inputs. If our signature is mostly `fn(Type) => Type`, isn't this the most weakened version?

One might wonder if I'm out of my damn mind. It looks like I'm proposing `lookupUser(User) => User`, without the caller knowing what's required to look up a user. How do we even use such a function? This is all crazy talk. At this point, you're demanding your money back, taking your lounging cactuar with you.

Why, my dear cactuar-loving reader, you've just helped segue into the next section!

## So what, then?

I may have rambled, but I promise you there's a simple point in the inscrutable wall of text above. The point can be made up into a nice kōan:

The type annotation `type?` we've used is made up of two things:
1. The value
2. How the value is used

We must split these apart into value and usage.

That's it.

Like all the hard problems of programming, this is a case of multiple things being put together, when they should be apart. And it's taking us 60 years to realise it.

Wouldn't it be amazing if we didn't have to ask whether something was there, because it was just there? Isn't this what we mostly want to talk about?
That was the beauty of using a `Maybe`: We could talk about values that *were*. Let's try to reclaim that experience.

Let's write a `bill` function. It can operate on a person's credit card, or if they don't have it on them, mail them a bill by their address.

### Overloading

Pseudo-code:
```js
function bill({ cc: { number, expiration, ccv } }) {
    // bill by cc
}

function bill({ address }) {
    // bill by address
}
```

Pros:
- Solves the problem at the call site
Cons:
- Spaces apart declarations, making order significant. Imagine if the two `bill` declarations were 1000 lines apart.

We weaken our inputs by providing another overload:

```js
function bill({ email }) {}
```

### Pattern-matching
A possible solution is *structural* pattern matching (pseudocode):

```js
function bill(user) {
    match(user) {
        ({ cc: { number, expiration, ccv }}) {
            // bill by cc
        },
        ({ address }) {
            // bill by address
        },
    }
}
```

Pros
- Explicit ordering
Cons
- Another flow-control construct

(The C#, Rust, Scala, Erlang, Haskell, ...and other people are now crying out that they have this. You don't really have this, sorry; see the appendix.)

Personally, I'm torn between these two solutions (multimethods and pattern matching and). The former *looks* more beautiful. The former *is* abstractly more beautiful. It fits into the language like a glove, and not *another thing*. It's not an `if` replacement, an orthogonal `switch...case`. It's just a function which accepts an argument.

In a language like Clojure, it would look even more beautiful(pseudo-clojure):

```clojure
(defn bill
  ([{ { num :number exp :expiration ccv :ccv } :cc }] ...)
  ([{ addr :addr }] ...))
```

Defining this syntactically for js most other C-based languages would be tricky, to say the least. Another star in the belt of lisp-languages.

## Conclusion

XXX

- I'm not bashing `Maybe`s
- I'm not bashing languages (except Java)
- Why I haven't talked about outputs/return values

Though if we're talking javascript, I'm excited for this [pattern matching proposal](https://github.com/tc39/proposal-pattern-matching).

## Appendix: Dispatching in different languages

This appendix is dedicated to some research into other languages. If you're not into that, you can safely ignore this section.

So, don't we already have pattern matching based on object structure? Not really.

Let's look at something close, Erlang (and by extension, Elixir):

```erlang
yay(Arg) ->
  case Arg of
    {x, X} -> X * 2;
    {y, Y} -> Y * 3
  end.

start() ->
  A = yay({ x, 10 }),
  B = yay({ y, 10 }),
  io:fwrite("hello world ~p ~p", [A, B]).
```

To my knowledge, one of the first prolific uses of pattern matching came from Prolog, which Erlang is an extension of. Haskell's pattern matching is similar. Looks awesome. Except this matches by index, and not by name.

Names are really, really, really important. I'm going to stub a lot of toes here, but I believe Haskell's data declarations are evil. Don't believe me? Pop quiz, what does this describe:

```haskell
data Node = Node Road Road | EndNode Road
data Road = Road Int Node
```

What does each `Road` of the `Node` mean? Binary tree nodes? Maybe a trie with costs? What's that `Int`? The road number? What's the matter, can't tell? *Need some more context*?
Here, have at it. This is an example taken straight from the amazing book [Learn You A Haskell](http://learnyouahaskell.com/functionally-solving-problems#heathrow-to-london). Imagine if in addition to types, you had names to tell you all of this extra info. Wow.

Just being the 2nd in a list is not enough to tell us what a thing is. Maps are an amazing concept. We should use them. Please, Haskell people, use more Record types!

For another example, let's look at C#. From [their documentation](https://docs.microsoft.com/en-us/dotnet/csharp/pattern-matching):

```c#
public static double ComputeArea_Version3(object shape)
{
    switch (shape)
    {
        case Square s when s.Side == 0:
        case Circle c when c.Radius == 0:
            return 0;

        case Square s:
            return s.Side * s.Side;
        case Circle c:
            return c.Radius * c.Radius * Math.PI;
        default:
            throw new ArgumentException(
                message: "shape is not a recognized shape",
                paramName: nameof(shape));
    }
}
```

That's a good example of solid pattern matching. You even have case predicates. It looks almost the same in Rust and other languages.
The downside is how like other pattern matching facilities (that I know of), we're once again talking about *names* and not *structure*. Compare the above to a structure-based pattern matching (pseudo-code):

```js
function computeArea(shape) {
    match (shape) {
        ({ side }) => side * side,
        ({ radius }) => radius * radius * Math.PI,
        _ => throw new TypeError();
    }
}
```

In my stupid syntax, we lost predicates, but we gained *structure*. Our argument doesn't have to be called a Square or Circle, it has to have a side or radius. You can pass in a `MyCircle` or a `PaintedCircle` or anything not fitting into any class hierarchy, because we don't care about what it's called or which classpath it loaded from, *we care that it's a friggin circle*.

Now. I think Clojure had a better idea with multimethods. Compare the code above to:

```clojure
(defmulti area :shape)
(defmethod area :square [s]
  (* (:side s) (:side s)))
(defmethod area :circle [c]
  (* (:radius c) (:radius c) Math/PI))

(area { :shape :square :side 4 })
(area { :shape :circle :radius 7 })
```

It's *so close*, but not *quite there*: To discern which method to call, you need to compare against some predicate (in our case, the result of `:shape`). And our predicate will have to be destructuring, which isn't a predicate.

To see this shortcoming more explicitly in a language which also got so close but not quite there, let's look at Scala. Scala actually has structural typing in its type system:

```scala
import scala.language.reflectiveCalls

def foo(bar: { def x(): Int; }) = {
  bar.x() * 2
}

object a {
    def x() = { 10 }
}
foo(a) // 20
```

Fantastic: We've got *exactly* what we wanted: A way to match against arguments! What can possibly go wrong?

```scala
def foo(bar: { def x(): Int; }) = {
  bar.x() * 2
}
def foo(bar: { def y(): Int; }) = {
  bar.y() * 3
}
```

Smash that compile button!
```
double definition:
def foo(bar: AnyRef{def x(): Int}): Int at line 5 and
def foo(bar: AnyRef{def y(): Int}): Int at line 8
have same type after erasure: (bar: Object)Int
```

*sigh*

Scala, unfortunately for us all, runs on the JVM, where there's type erasure. To Scala, these are both just `Object`s, and we use object reflection to get to the properties after shaping.

I'm not a Scala man, but my gut tells me there's a solution using reflection. That (and Clojure multimethods) can probably look (roughly) like:

```js
function foo(obj) {
    if (looksLike(['x'], obj)) {
        // ...
    }
    else if (looksLike(['y'], obj)) {
        // ...
    }
}
```

## Appendix: TypeScript anonymous objects

A lot of the types presented here could be nicer when laid out with anonymous types. If we have our partial `Person` like:

```ts
type Person = {
    name: string,
    cc?: CreditCard,
    address?: string,
};
```

Then one can guarantee the presence of `cc` by writing:

```ts
function bill(Person: Person & { cc: CreditCard }) {}
```

To repeat earlier arguments as to why this is not good enough:
- Proliferates names and not structure (I bet you $5 some info on cc is optional)
- Duplicates type and usage - we want to destructure, we have to do it both in the value and the type
- Everything in the base type turns out `null`able/`Maybe`
- This still *does not* describe `billByCC` or `billByAddress`, just one.

Describing both?

```ts
function bill(Person: (Person & { cc: CreditCard }) | (Person & { address: string })) {}
```

And you end up not knowing which one you got, until you write that `or` in code as well. We've circled around to where we started, having to explicitly acknowledge and deal with missing values.