# Require acknowledgement of protocol extension dispatch rules

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
protocol it conforms to. I propose that we add mandatory keywords to 
both statically-dispatched protocol extension members and conflicting 
protocol conformances so that this behavior is reflected in the source 
code.

## Motivation

In Swift 2, a protocol extension can either provide a default 
implementation for a member defined in the protocol, or provide a new 
method which is not listed in the original protocol at all.

	protocol Turnable {
		func turning() -> Self
		mutating func turn()
	}
	extension Turnable {
		// This is a default implementation of `turn` because it is also
		// listed in the protocol.
		mutating func turn() {
			self = turning()
		}
		
		// This is an extension method. Note that it is not listed in the
		// protocol.
		func turningRepeatedly(additionalTurns: Int) -> Self {
			var turnedSelf = self
			for _ in 1...additionalTurns {
				turnedSelf.turn()
			}
			return turnedSelf
		}
	}

A conforming type *must* provide implementations of any member listed in
the protocol which are not defined in any extension. They *may* provide 
implementations of members which have default values; in that case, the 
type's implementation silently overrides the protocol extension's. The 
type may also silently non-default extension methods, that is, ones 
which are not listed in the original protocol...but there's a catch.

	class SpimsterWicket: Turnable {
		var turns: Int = 0
		
		required init(turns: Int = 0) {
			self.turns = turns
		}
		
		// Note that the default implementations in Turnable assume value
		// semantics. Our SpimsterWickets have reference semantics, so those
		// implementations won't work. We'll have to provide our own.
		
		func turn() {
			turns += 1
		}
		
		func turning() -> Self {
			return self.dynamicType.init(turns: turns + 1)
		}
		
		func turningRepeatedly(additionalTurns: Int) -> Self {
			return self.dynamicType.init(turns: turns + additionalTurns)
		}
	}

In testing, this all works fine as long as we're working with a variable
of type `SpimsterWicket`. But if we use our `SpimsterWicket` as a `Turnable`,
either by casting it to `Turnable` or by passing it to a generic function
constrained to `Turnable`, then `Turnable`'s implementation of 
`turningRepeatedly` is called, which in this case gives the wrong 
semantics.

	func turn5<T: Turnable>(input: T) -> T { 
		return input.turningRepeatedly(5) 
	}
	
	let input = SpimsterWicket()
	turn5(input)
	print(input.turns)		// prints "5"

## Proposed solution

I do not propose that we change the dispatch semantics. Rather, I propose
we make this behavior more obvious to language users in two separate, 
but interlocking, ways.

The first is that any member which is in a protocol extension but not in 
the protocol itself—that is, any member which does not participate in 
dynamic dispatch—should be required to be marked `final`. Failing to 
mark such a member `final` should be an error, and a fix-it to add the 
`final` keyword should be provided.

The second is that having both a protocol extension containing a final 
member, and a type conforming to that protocol with a member with the 
same name and signature, visible in the same source file should normally 
be an error. The preferred means of fixing this error would be to rename
one of the conflicting members. However, if the developer can't or 
doesn't want to do so, an `@incoherent` attribute should be applied to 
the declaration which causes the conflict. Because renaming is preferred,
a fix-it adding an `@incoherent` attribute should *not* be offered.
[XXX If we can offer renaming fix-its and list them above adding 
`@incoherent`, I think we could have a fix-it adding `@incoherent` too.]

Since a conflict of this sort can be created in several ways, an 
`@incoherent` attribute can be applied to either the conformance, the 
protocol extension, or the `import` statement which causes the conflict:

	class SpimsterWicket: @incoherent Turnable {...
	
	@incoherent(SpimsterWicket) extension Turnable {...
	
	import TurnableKit
	import LibWicket
	@incoherent(SpimsterWicket: Turnable) import TurnableExtensions

The `@incoherent` attribute should always be specific to a particular 
combination of type and protocol. For a conformance, both are obvious
from context. For a protocol extension, the type name should be 
indicated in parentheses. For an `import` statement, both type and 
protocol should be indicated, separated by a colon. In the latter two 
cases, several conflicts can be listed in a single attribute by 
separating them with commas:

	@incoherent(SpimsterWicket, Phonograph) extension Turnable {
	@incoherent(SpimsterWicket: Turnable, Phonograph: Turnable) import TurnableExtensions

The error for a particular conflict should only be shown on one of the
involved declarations, and always in the current module. Where there's 
more than one involved declaration in the current module, Swift should
select the first of these available:

1. The conformance
2. The protocol extension
3. The last relevant `import` line in each file which can see the conflict

Thus, if both the conformance and the protocol extension are in the 
current module, the error should be shown on the conformance. If only 
the protocol extension is in the current module, the error should be on
the protocol extension. If neither is in the current module—the 
conformance comes from module `A` and the protocol extension comes from 
module `B`—then the error should be shown on whichever of `import A` or 
`import B` comes last in files which can see both.

## Detailed design

(TBD, unless the above is specific enough.)

## Impact on existing code

The requirement that `final` be applied to many protocol extension 
methods will cause many—perhaps most—protocol extensions to stop 
compiling without changes. However, the changes needed—adding a keyword 
to the declarations of relevant members—are fairly mechanical and are 
fix-it guided.

The requirement that conflicting conformances be flagged with an 
attribute will cause a smaller number of conformance declarations, 
protocol extensions, or `import` statements to fail to compile. I 
believe that many of these cases will be bugs, and users will want to
rename members to avoid them. When users do not want to rename, 

## Alternatives considered

### Prevent having a conflicting protocol extension and conformance visible, with no escape hatch.

This would improve safety by making this confusing construct completely 
impossible to write. However, it would also make it completely 
impossible to conform certain types to certain protocols or import 
certain combinations of modules into the same file. This seems especially 
unwise because previous versions of Swift have actually permitted this 
shadowing; code that previously compiled could be difficult to port.

### Dynamically dispatch calls to protocol extension members.

This would likely take a redesign of protocol witnesses to include 
extension methods not listed in the original protocol. It's not 
impossible—class extensions behave this way—but it's a much bigger
change than what I propose, which keeps the current semantics but 
requires additional keywords to acknowledge them.

Dynamically dispatching to protocol extension members would also change
the performance characteristics of these calls. Even if this change 
were made, we might want to allow users to apply `final` to extension 
methods which they want to be dispatched statically.

### Mark default members instead of statically dispatched members.

This would invert the keywording in a protocol extension: instead of 
marking the statically-dispatched methods with `final`, you would mark 
the overridable methods with `default`.

I prefer `final` for some fairly soft reasons: the static behavior is 
the surprising behavior; Swift usually favors making dynamic behavior,
where it exists, opt-out instead of opt-in (which is why it has `final` 
instead of `virtual`); and I could see the use of `final` being retained
even if dynamic dispatch of these calls is added later.

However, my preference is not terribly strong, and using `default` on the
overridable methods is not a bad option.

### Only mark statically-dispatched extension members; don't mark conflicting conformances.

### Only mark conflicting conformances; don't mark statically-dispatched extension members.

