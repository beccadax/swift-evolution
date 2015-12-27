# Require `final` on protocol extension members

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): [Brent Royal-Gordon](https://github.com/brentdax)
* Status: **Draft**
* Review manager: TBD

## Introduction

Protocol extension members which aren't listed in the protocol itself 
have an unusual behavior: a conforming type can implement an identically
named member, but instances with the protocol's type are always 
statically dispatched to the protocol's implementation. This can lead to
the same instance displaying different behavior when it's cast to a 
protocol it conforms to. In effect, the conforming type's member shadows
the protocol's, rather than overriding it. This behavior is very 
surprising to some users.

The lack of a warning on this is [currently considered a bug] (https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001861.html), but I think we should go further and cause it to be an error. However, we should also provide an escape hatch which permits conflicts in cases where they're necessary.

## Motivation

Suppose you write a protocol and extension like this:

	protocol Turnable {
	    func turning() -> Self
	    mutating func turn()
	}
	extension Turnable {
	    mutating func turn() {
	        self = turning()
	    }
	
	    func turningRepeatedly(additionalTurns: Int) -> Self {
	        var turnedSelf = self
	        for _ in 1...additionalTurns {
	            turnedSelf.turn()
	        }
	        return turnedSelf
	    }
	}

Now you want to write a conforming type, `SpimsterWicket`. There are
three different rules about whether your type has to, or can, implement
its own versions of these methods.

1. `turning()` is a “protocol method”: it is listed in the protocol
    but is not included in the extension. You *must* implement
    `turning()` to conform to `Turnable`.
2. `turn()` is a “defaulted protocol method”: it is listed in the
    protocol but there is also an implementation of it in the
    extension. You *may* implement `turn()`; if you don’t, the
    protocol extension’s implementation will be used.
3. `turningRepeatedly(_: Int)` is a “protocol extension method”: it
    is *not* listed in the protocol, but only in the protocol 
	extension. This is the case we are trying to address.

Currently, in case 3, Swift permits you to implement your own 
`turningRepeatedly(_: Int)`. However, your implementation may not be 
called in every circumstance that you expect. If you call 
`turningRepeatedly` on a variable of type `SpimsterWicket`, you’ll 
get `SpimsterWicket`’s implementation of the method; however, if you 
call `turningRepeatedly` on a variable of type `Turnable`, you’ll get
`Turnable`’s implementation of the method.

	var wicket: SpimsterWicket = SpimsterWicket()
	var turnable: Turnable = wicket
	
	wicket.turn()					// Calls SpimsterWicket.turn()
	turnable.turn()					// Also calls SpimsterWicket.turn()
	
	wicket.turningRepeatedly(5)		// Calls SpimsterWicket.turningRepeatedly(_:)
	turnable.turningRepeatedly(5)	// Calls Turnable.turningRepeatedly(_:)

In most parts of Swift, casting an instance or assigning it to a 
variable of a different type doesn’t change which implementation will
be called when you put it on the left-hand side of a dot. (I’m 
leaving aside Objective-C bridging, like `Int` to `NSNumber`, which is
really a different operation being performed with the same syntax.) If
you put a `UIControl` into a variable of type `UIView`, and then call
`touchesBegan()` on that variable, Swift will still call 
`UIControl.touchesBegan()`. The same is true of defaulted protocol
methods—if you call `turn()` on `turnable`, you’ll get `
SpimsterWicket.turn()`.

But this is not true of protocol extension methods. There, the static 
type of the variable—the type known at compile time, the type that 
the variable is labeled with—is used. Thus, calling 
`turningRepeatedly(_:)` on `wicket` gets you `SpimsterWicket`’s
implementation, but calling it on `turnable`—even though it's merely
the same instance casted to a different type—gets you `Turnable`’s
implementation.

This creates what I call an “incoherent” dispatch, and it occurs
nowhere else in Swift. In most places in Swift, method dispatch is 
either based on the runtime type (reference types, normal protocol 
members), or the design of the language ensures there’s no difference
between dispatching on the compile-time type and the runtime type
(value types, `final` members). But in protocol extension members,
dispatch is based on the compile-time type even though the runtime type
might produce different behavior.

## Proposed solution

I propose that we:

1. Cause Swift to emit an error when it detects this sort of shadowing.
2. Add a mandatory `final` keyword to statically-dispatched protocol 
   extension members, to give a textual indication that these errors will
   occur.
3. For those circumstances in which this shadowing is a necessary evil,
   provide an attribute which can be applied to indicate that conflicts 
   are allowed when caused by a particular conformance.

Specifics follow, though not in the order given above. In the examples 
below, `T` and `U` are conforming types, `P` and `Q` are protocols, and
`f` is a member which, in some cases, may be a final protocol extension
member.

### Mark protocol extension members with the `final` keyword

If we are going to emit an error for these conflicts, it would be 
helpful to mark which members are prone to them.

I therefore propose that all protocol extension members (that is, the 
ones that aren't providing a default implementation for a protocol 
requirement) be marked with the `final` keyword. Failing to mark such a
member with `final` would cause an error.

Currently, the `final` keyword is only used on classes. When applied to 
a class's member, it indicates that subclasses cannot override that 
member. I see this new use of the `final` keyword as analogously 
indicating that conforming types cannot customize the member. In fact, 
you can capture both meanings of `final` in a single statement:

> `final` declares that subtypes of this type must use this specific 
> implementation of the member, and cannot substitute their own 
> specialized implementation. Attempting to do so causes an error.

### Make conflicting with a `final` protocol extension method an error

We can now define an incoherent type as one in which a `final` protocol
extension member is shadowed by a member of a conforming type, as seen
from any source file. (Note that methods and subscripts are only 
shadowed by a member with a matching signature.)

Because incoherence is caused by the interaction of two separate 
declarations, there are many different circumstances in which 
incoherence may be caused. Here are the ones I've been able to think 
of:

1. Type `T` is conformed to protocol `P` in a file where member `f` is
   visible on both `T` and `P`.
2. Type `T` is conformed to protocols `P` and `Q`, which both have a
   final member `f`. (Note that diamond conformance patterns, where `P`
   and `Q` both get the same final member `f` from protocol `R`, should
   be permitted.)
3. Type `T` is extended to add a member `f` in a file where `T`'s
   conformance to `P` is imported from another module, and `P` has a
   final member `f`.
4. Protocol `P` is extended to add final member `f` in a file where
   `T`'s conformance to `P`, and the declaration of `T.f`, are both
   imported from other modules.
5. A source file imports module A, which extends protocol `P` to
   include final member `f`, and module B, which conforms type `T` with
   member `f` to conform to `P`.

It should be noted that these conflicts are tied to *visibility*. There
is no conflict if the two definitions of `f` are not both visible in 
the same place. For instance:

- If file A.swift extends `T` with a private member `f`, and file 
  B.swift extends `P` with a private final member `f`, there is no
  conflict.
- If module A extends `T` with an internal member `f`, and module B
  extends `P` with an internal final member `f`, there is no conflict.
- If module A extends `T` with a *public* member `f`, and module B
  extends `P` with a public final member `f`, there is only a conflict
  if A and B are imported in the same file. Even if A and B are
  imported in different files in the same module, there is no conflict.

### Permit conflicts with an explicit acknowledgement through an `@incoherent` attribute

In some circumstances, it may be desirable to permit a conflict, even
though it causes surprising behavior. For instance, you may want to
conform an existing type to a protocol where the names conflict through
sheer happenstance, and you know the protocol extension method will 
only ever be needed in code that treats that uses the protocol's type.
In those cases, you can disable the conflict error and restore the
current incoherent dispatch behavior using an `@incoherent` attribute.

The `@incoherent` attribute is always tied to a particular type and 
protocol. It says, in essence, "I know type T conflicts with protocol 
P, and I want to ignore all of those conflicts and accept incoherent
dispatch." Depending on where it's attached, you may have to specify
more or less information in the attribute's parameters. For instance:

	// Mark the conformance.
	extension T: @incoherent P {...}
	
	// Mark the extension.
	@incoherent(T) extension P {...}
	@incoherent(P) extension T {...}
	
	// Mark the import statement
	@incoherent(T: P) import B

## Detailed design

### Errors for improper use of the `final` keyword

Failing to put a `final` keyword on a protocol extension member which
requires it should emit an error message along these lines:

    f must be final because it is not a requirement of P.

This error should include a fix-it which adds the `final` keyword.

Putting a `final` keyword on a defaulted protocol member is 
nonsensical—it essentially gives the member the semantics of a 
protocol extension member. We should emit an error message along these
lines:

    f cannot be final because it is a requirement of P.

This error should include a fix-it which removes the `final` keyword.

### Errors for conflicting members

As mentioned above, there are many ways to cause a conflict, and each
of them needs a slightly different wording. Here's what I propose:

1. **Type `T` is conformed to protocol `P` in a file where member `f`
   is visible on both `T` and `P`.** The declaration of the conformance
   (that is, the declaration with the `: P` clause) should be marked
   with an error like:

       T cannot conform to P because T.f conflicts with final member P.f.
	
2. **Type `T` is conformed to protocols `P` and `Q`, which both have a
   final member `f`.** The declaration of one of the conformances
   should be marked with an error like:

       T cannot conform to both P and Q because final member P.f conflicts with final member Q.f.

3. **Type `T` is extended to add a member `f` in a file where `T`'s
   conformance to `P` is imported from another module, and `P` has a
   final member `f`.** The declaration of the concrete type extension
   should be marked with an error like:

       T cannot be extended to add member f because it conflicts with final member P.f.

4. **Protocol `P` is extended to add final member `f` in a file where
   `T`'s conformance to `P`, and the declaration of `T.f`, are both
   imported from other modules.** The declaration of the protocol
   extension should be marked with an error like:

       P cannot be extended to add final member f because it conflicts with member T.f of a conforming type.
	
5. **A source file imports module A, which extends protocol `P` to
   include final member `f`, and module B, which conforms type `T` with
   member `f` to conform to `P`.** The later of the two imports should
   be marked with an error like:

       B cannot be imported because final member P.f conflicts with A's T.f.

The preferred means of resolving a conflict include:

- Renaming one of the conflicting members.
- Deleting one of the conflicting members.
- Deleting an `import` statement which causes the conflict.
- Adding the conflicting member to the protocol, and removing the 
  `final` keyword, so that it becomes a defaulted protocol member.

If it is feasible to provide a fix-it suggesting one of these s
olutions, that should be done.

Fix-its should *not* suggest adding an `@incoherent` attribute. It is
sometimes necessary to permit incoherence, but it's never desirable,
because incoherence is confusing. A fix-it would encourage users to 
enable incoherence without actually understanding what it means, which
will cause confusion.

### Marking multiple `@incoherent` types

In places where one parameter to the `@incoherent` attribute is 
permitted, you can instead provide a comma-separated list to permit 
several different incoherences caused by the same declaration:

	@incoherent(T, U) extension P {...}
	@incoherent(P, Q) extension T {...}
	
	@incoherent(T: P, U: Q) import B

## Impact on existing code

The requirement that `final` be applied to many protocol extension 
methods will cause many—perhaps most—protocol extensions to stop 
compiling without changes. However, the changes needed—adding a keyword 
to the declarations of relevant members—are fairly mechanical and are 
fix-it guided. The migrator should add them automatically.

The conflict errors will cause a smaller number of conformance 
declarations, protocol extensions, or `import` statements to fail to 
compile. I believe that many of these cases will be bugs, and users 
will want to rename members to avoid them. When users do not want to 
rename, they can preserve the existing semantics with an `@incoherent`
attribute.

Because this is primarily a safety feature, does not change runtime 
semantics, and, in most codebases, all changes will be purely 
mechanical, I believe this feature should be added to the next minor 
version of Swift, rather than waiting for 3.0.

## Alternatives considered

### Dynamically dispatch calls to protocol extension members

This would fix the underlying problem—the confusing behavior—by
making protocol extension members not behave confusingly.

This would likely take a redesign of protocol witnesses to include 
extension methods not listed in the original protocol. It's probably
not impossible—class extensions behave this way—but it's a much 
bigger change than what I propose, which keeps the current runtime 
semantics and only adds compile-time errors and keywords.

Dynamically dispatching to protocol extension members would also change
the performance characteristics of these calls. Even if this change 
were made, we might want to allow users to apply `final` to extension 
methods which they want to be dispatched statically.

### Don't provide an `@incoherent` attribute

This would improve safety by making this confusing construct completely 
impossible to write. However, it would also make it completely 
impossible to conform certain types to certain protocols or import 
certain combinations of modules into the same file. This seems especially 
unwise because previous versions of Swift have actually permitted this 
shadowing; code that previously compiled without even a warning could 
be difficult to port.

### Mark default members instead of statically dispatched members

This would invert the keywording in a protocol extension: instead of 
marking the statically-dispatched members with `final`, you would mark 
the overridable members with `default`.

I prefer `final` because it marks the more unusual case. Users are not 
surprised that they can override default methods; they are surprised 
that they *can't* reliably override protocol extension methods. Also, 
as mentioned in the previous section, I could see `final` being 
retained even if protocol extension methods gained dynamic dispatch.

However, my preference is not terribly strong, and using `default` on the
overridable methods is not a bad option.

### Require a `final` keyword, but don't prevent conflicts with an error

The problem with this approach is that the conflict is the surprising 
part. It doesn't matter all that much whether protocol extension members 
are dispatched statically or dynamically *except* if there's a conflict;
*then* you're getting into potential bug territory. The `@incoherent` 
keyword is what makes this mistake impossible to make accidentally; 
without it, this is merely a proposal to force people to annotate their 
protocol extensions more clearly.

### Don't require a `final` keyword, but prevent conflicts with an error

Without the `final` keyword (or the `default` alternative mentioned 
above) on the extension members themselves, it's impossible to tell at 
a glance which members are overridable and which ones aren't. This makes
predicting incoherent conformance errors an exercise in trial-and-error,
or at least in visual-diffing two separate parts of the code to figure 
out what's going on. Without the `final` keyword, in other words, 
avoiding conflicts when you write your code instead of discovering them 
when you compile it is much more difficult.
