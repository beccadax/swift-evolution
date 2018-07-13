# Allow operators to have additional defaulted parameters

* Proposal: [SE-NNNN](NNNN-default-parameters-ops.md)
* Authors: [Brent Royal-Gordon](https://github.com/brentdax)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

Swift includes several features which let functions automatically collect
information about the call site, such as file and line number, by adding
parameters with default values. We propose extending this to operators by
allowing them to declare additional parameters, as long as the extras have
default values.

Swift-evolution thread: [Proposal: Allow operators to have parameters with default values](https://forums.swift.org/t/proposal-allow-operators-to-have-parameters-with-default-values/6937/2)

## Motivation

Swift functions sometimes want to capture information about the place
where they were called. For example, `Swift.precondition` includes the
file and line of the failing precondition; `XCTAssert` and friends
indicate the file and line number; `os_log` needs a pointer to the
object file the caller resides in.

This is made slightly more complicated because some functions need to
call these functions, but emit an error as though *their* caller had
called them. For example, an analytics package might add a message
to its log, then call through to `Swift.assert`.

Many languages build heavyweight runtime-based call stack introspection
features to allow this. In contrast, Swift uses an elegant mechanism which
costs nothing when you don't need it: defaulted parameters. A default
parameter value which uses one of the magic `#file`, `#function`, `#line`,
`#column`, or `#dsohandle` literals expands it as though it was provided at
the call site, so functions which need this information can capture it invisibly
without it needing to be tracked for functions which don't. The use of default
parameters also makes it easy to wrap one of these functions: simply capture
the information from your own caller through a default parameter, then specify
the value of that parameter for your callee instead of accepting the default
value.

One example of these features in action comes from the standard library's
internal precondition functions:

```swift
@_transparent
public func _precondition(
  _ condition: @autoclosure () -> Bool, _ message: StaticString = StaticString(),
  file: StaticString = #file, line: UInt = #line
) {
  // implementation omitted
}

@_transparent
public func _preconditionFailure(
  _ message: StaticString = StaticString(),
  file: StaticString = #file, line: UInt = #line
) -> Never {
  _precondition(false, message, file: file, line: line)
  _conditionallyUnreachable()
}
```

Swift also supports custom operators, which are basically a form of
function with non-identifier names and a separate call syntax. However,
operators have a fixed number of parameters (one for prefix and postfix,
two for infix) and it is a compiler error to declare one which takes any
more than that, even if they have default values. This prevents operators
from capturing call-site information in the way functions do, which in turn
makes them less appealing for features like assertions, logging, and testing
which benefit from capturing call site information.

The recent [SE-0217][bangbang] proposed such an operator, and enthusiasm
for the proposal was substantially dampened by its inability to capture call-site
information. Whether or not that operator belongs in the standard library, Swift
users who want to implement it—or others like it—themselves should be able
to use a fully-featured version of it. They can't without this minor language
change.

  [bangbang]: https://github.com/apple/swift-evolution/blob/master/proposals/0217-bangbang.md

## Proposed solution

We propose allowing prefix and postfix operators to take more than one
parameter, and infix operators to take more than two parameters, as
long as all additional parameters have default values. Defining such an
operator would look something like this:

```swift
operator ++= : AssignmentPrecedence

/// Appends to an NSAttributedString, annotating it with debug attributes
/// describing where in the code the appending was done.
func ++= (lhs: NSMutableAttributedString, rhs: NSAttributedString,
          file file: String = #file, line line: Int = #line) {
  let startIndex = lhs.length
  lhs.append(rhs)
  lhs.addAttributes([.debugSourceFile: file, .debugSourceLine: line],
    range: NSMakeRange(startIndex, length))
}
```

Use would look no different from a custom operator which doesn't capture call
site information:

```swift
allCode ++= codeFragment
```

Callers which need to provide non-default arguments for the extra parameters
could use the existing function-style operator syntax:

```swift
extension CodeTemplate {
  func append(to allCode: NSMutableAttributedString,
              file: String = #file, line: Int = #line) {
    (++=)(allCode, makeCode(), file: file, line: line)
  }
}
```

This syntax admittedly won't win any beauty contests, but we don't believe it
will be needed very often. If the author of a particular operator anticipates
that many callers will need to override the defaults, they can always provide an
overload which (say) takes a tuple on the right-hand side containing both the
operand and the extra parameters.

## Detailed design

Currently, operator parameters are always unlabeled. Additional defaulted
parameters will be unlabeled by default, but can specify labels explicitly,
much like subscript parameters. These labels are only used when specifying
non-default arguments for the parameters using the function-style syntax.

Users will not be restricted to only using the magic identifiers as default
values—they can specify any expression that would be valid in a function
parameter's default value. We don't think this will actually be useful, but we
don't think preventing it is worth complicating the compiler. If someone comes
up with a way to use it, more power to them.

### Implementation

The implementation of this feature is fairly straightforward and largely
makes unary and binary operators more similar to ordinary function calls
in the AST. The most impactful change is that some parts of the compiler
currently assume that the argument to an operator is an unparenthesized
expression or a two-element tuple expression; these need to be modified
to handle the full generality of an argument list, including the tuple
shuffles used to implement default values.

This change requires the constraint solver to do a more "correct" match
of operator arguments to parameters, rather than assuming their arity.
[[TODO: Does this affect compiler performance?]]

## Source compatibility

Any attempt to declare an operator with extra defaulted parameters was
previously an error, so this proposal should be purely additive.

The changes to AST representation and operator overload resolution could,
in theory, cause subtle source incompatibilities. [[TODO: Make sure there
aren't any in the source compatibility suite.]] If necessary, we could
stick to the old behavior in Swift 4 mode, at the cost of making operators
with extra parameters invisible there.

## Effect on ABI stability

This change does not affect the ABI of any currently existing symbols,
but if any operators in the standard library wanted to capture call
context, that would be an ABI-affecting change.

## Effect on API resilience

Adding extra parameters to an existing operator would be binary-incompatible
but source-compatible. This is a special case of the general rule that adding
defaulted parameters to a function is binary-incompatible but source-compatible.

## Alternatives considered

Some people suggested that macros might obviate the need for this proposal.
However, we don't know when or if macros will be added, we don't know if they
will support this use case (they might require you to mark operators
implemented by macros at the use site), and we don't know if mere mortals will
be able to define them easily. The default-parameter-based feature for
functions is already supported and not going away; extending it to operators is
a very natural solution that requires little additional work.

A few people suggested adding a new mechanism to reflect on the call stack
at runtime. We think that would be killing two birds with one howitzer. Call
stack reflection would be a very complicated feature, and it's not clear we
can do it without unacceptable costs. It also would require a new way to skip
the call stack frames of wrapper functions. And a failure-reporting function
seems like a dubious place to be reflecting on the call stack.

We could add some side channel to pass this information instead of using
parameters, or introduce some new syntax which translated into parameters, but
we don't see much benefit. The parameter list is already there and already
serves this role for functions.

We could package up all available magic-identifier information into a single
magic identifier which returned a tuple or struct. We think that change would
be best considered as a separate proposal.

## Future directions

If a future proposal added a decent-looking syntax to supply values for extra
parameters, we could define operators which could be parameterized. For
instance, with the straw syntax `#opts()`:

```swift
if people[i].name > people[j].name #opts(caseInsensitive: true) {
  people.swapAt(i, j)
}
```

We are not planning to make such a proposal, but thought we'd mention it
for the sake of completeness.
