# Throwing Properties and Subscripts

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): [Brent Royal-Gordon](https://github.com/brentdax)
* Status: **Draft**
* Review manager: TBD

## Introduction

Functions, methods, and initializers can be marked `throws` to indicate 
that they can fail by throwing an error, but properties and subscripts 
cannot. This proposal extends properties and subscripts to support 
`throws` and `rethrows` accessors, and also specifies logic for 
bridging these accessors to and from Objective-C.

Swift-evolution thread: [Proposal: Allow Getters and Setters to Throw](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001165.html)

## Motivation

Sometimes, something that is genuinely getter- or setter-like needs to 
be able to throw an error. This is particularly common with properties
which convert between formats:

```swift
var image: UIImage

var imageData: NSData {
	get {
		return UIImagePNGRepresentation(image)
	}
	set {
		image = UIImage(data: newValue) ?? throw OopsICantThrowHere
	}
}
```

Or which access some external resource which may not be able to perform 
the operation:

```swift
var avatar: UIImage {
	get {
		let data = try NSData(contentsOfURL: avatarURL, options: [])	/* can't try here! */
		return UIImage(data: data)
	}
}
```

The current best solution to this problem is to write a method instead 
of a property. This can lead to unnatural API designs; the class
`AVAudioSession` alone, for instance, has no less than ten mismatched 
property/setter method pairs:

```swift
var category: String { get }
func setCategory(_ category: String) throws

var mode: String { get }
func setMode(_ mode: String) throws

var inputGain: Float { get }
func setInputGain(_ gain: Float) throws

var preferredSampleRate: Double { get }
func setPreferredSampleRate(_ sampleRate: Double) throws

var preferredIOBufferDuration: NSTimeInterval { get }
func setPreferredIOBufferDuration(_ duration: NSTimeInterval) throws

var preferredInputNumberOfChannels: Int { get }
func setPreferredInputNumberOfChannels(_ count: Int) throws

var preferredOutputNumberOfChannels: Int { get }
func setPreferredOutputNumberOfChannels(_ count: Int) throws

var preferredInput: AVAudioSessionPortDescription? { get }
func setPreferredInput(_ inPort: AVAudioSessionPortDescription?) throws

var inputDataSource: AVAudioSessionDataSourceDescription? { get }
func setInputDataSource(_ dataSource: AVAudioSessionDataSourceDescription?) throws

var outputDataSource: AVAudioSessionDataSourceDescription? { get }
func setOutputDataSource(_ dataSource: AVAudioSessionDataSourceDescription?) throws
```

While most classes aren't nearly this bad, you see the same problem 
elsewhere in the frameworks. The Mac-only `CoreWLAN` framework has 
similar mismatched property/setter method pairs (though it also has 
other bridging issues; I suspect it's too obscure to have been audited 
yet):

```swift
func wlanChannel() -> CWChannel!
func setWLANChannel(_ channel: CWChannel!, error error: NSErrorPointer) -> Bool

func powerOn() -> Bool
func setPower(_ power: Bool, error error: NSErrorPointer) -> Bool
```

When the getter can throw, it gets even worse. `NSURL` has an awkward 
pair of methods to get "resource values" which would be better 
expressed as a throwing read-write subscript:

```swift
func getResourceValue(_ value: AutoreleasingUnsafeMutablePointer<AnyObject?>, forKey key: String) throws
func setResourceValue(_ value: AnyObject?, forKey key: String) throws
```

## Proposed solution

Swift can handle these cases better by allowing getters and setters to
throw.

### Throwing computed properties

You can mark a computed property accessor as throwing by putting 
`throws` after the `get` or `set` keyword:

```swift
var property: Int {
	get throws { ... }
	set throws { ... }
}

subscript(index: Int) -> Bool {
	get throws { ... }
	set throws { ... }
}
```

The throwing behavior of the getter and setter are completely 
independent; a throwing getter can be paired with a non-throwing 
setter, or vice versa.

```swift
var property: Int {
	get throws { ... }
	set { ... }
}

subscript(index: Int) -> Bool {
	get { ... }
	set throws { ... }
}
```

A protocol (or, if added later, an abstract class) can specify the 
throwing behavior of properties and subscripts it requires:

```swift
protocol MyProtocol {
	var property: Int { get throws set throws }
	subscript(index: Int) -> Bool { get throws set throws }
}
```

### Throwing stored properties

A stored property can also be given a throwing setter by giving it a 
`willSet` accessor that throws:

```swift
var property: Int {
	willSet throws {
		guard newValue >= 0 else {
			throw MyError.PropertyOutOfRange (newValue)
		}
	}
}
```

### Importing throwing property accessors from Objective-C

When a readonly property `foo` of type `T` is imported from Objective-C,
but a method like this exists on the same type:

```objc
- (BOOL)setFoo:(T)value error:(NSError**)error;
```

Swift will import `foo` as a readwrite property with a throwing setter.

If [SE-0044 Import as member](https://github.com/apple/swift-evolution/blob/master/proposals/0044-import-as-member.md)
is accepted, we should also be able to apply the `swift_name` attribute 
to methods of these forms to create properties with throwing getters:

```objc
- (nullable T)foo:(NSError**)error;		// property is not optional
- (BOOL)getFoo:(T*)outValue error:(NSError**)error;
```

No imports for throwing subscript accessors are specified.

These transformations should be applied to both classes and protocols.

### Exporting throwing property and subscript accessors to Objective-C

A throwing setter for a property `foo` of type `T` should be exposed to
Objective-C as:

```objc
- (void)setFoo:(T)value error:(NSError**)error;
```

A throwing getter for a property `foo` of type `T`, where `T` is not 
optional but can be nullable in Objective-C, should be exposed to 
Objective-C as:

```objc
- (nullable T)foo:(NSError**)error;
```

Otherwise, the getter should be exposed as:

```objc
- (BOOL)getFoo:(nonnull T*)outValue error:(NSError**)error;
```

A throwing setter for a subscript of type `T` with an index of type 
`I`, if marked with `@objc(name)`, should be compatible with this 
signature:

```objc
- (BOOL)setFoo:(T)value atIndex:(I)index error:(NSError**)error;
```

A throwing getter for a subscript of type `T` with index `I`, where 
`T` is not optional but can be nullable in Objective-C, should be 
compatible with this signature:

```objc
- (nullable T)fooAtIndex:(I)index error:(NSError**)error;
```

Otherwise, the getter should be have a name compatible with this 
signature:

```objc
- (BOOL)getFoo:(nonnull T*)outValue atIndex:(I)index error:(NSError**)error;
```

Throwing subscript accessors which are not marked with `@objc(name)` 
will not be exposed to Objective-C.

These transformations should be applied to both classes and `@objc` protocols.

## Detailed design

### Subscripts with `rethrows`

`rethrows` is not supported on properties, but it is supported on 
subscripts. The rethrowing behavior depends only on the subscript's  
parameters, not the setter's `newValue`; that is, a particular 
subscript access can throw iff at least one of the functions inside the 
square brackets can throw.

### Throwing accessors and `inout` parameters

A throwing property or subscript access can be passed as an `inout` 
parameter. The call it is passed to must be marked with the `try` 
keyword.

To avoid unpredictable interactions between `inout` and throwing 
accessors, Swift will guarantee the getter is invoked once before the 
call and the setter once after the call. The compiler will not apply 
optimizations which might cause errors to be thrown in the middle of 
the function.

### Throwing requirement compatibility

An implementation can be "less" throwing than a requirement it is 
intended to satisfy. That is:

* A throwing accessor requirement can be fulfilled by a throwing, 
  rethrowing, or non-throwing accessor.
* A rethrowing accessor requirement can be fulfilled by a rethrowing
  or non-throwing accessor.
* A non-throwing accessor requirement can be fulfilled only by a 
  non-throwing accessor.

These definitions apply to protocol (and abstract class) conformance, 
subclass overrides, and library resilience. (Note that last point: 
Swift must permit an accessor to be made less throwing without breaking 
binary compatibility.)

When overriding a throwing accessor, the override must explicitly state
the expected level of throwing behavior; omitting the keyword means the
accessor is non-throwing. That is, in this example, `Subclass.foo`'s
setter is not automatically `throws`:

	class Superclass {
		var foo: Int {
			willSet throws { ... }
		}
	}
	
	class Subclass: Superclass {
		override var foo: Int {
			set { try super.foo = newValue }
			// Error: nonthrowing setter includes throwing statement
		}
	}

### Implementation

The internal `materializeForSet` is as throwing as the "most" throwing 
of `get` and `set`.

**FIXME**: Beyond that, I have no idea. Sorry. Please help me fill this 
out.

## Impact on existing code

Some APIs will be imported differently, breaking call sites. The Swift 
compiler will need to provide fix-it and migration support for these 
cases.

## Alternatives considered

### Require setters to be at least as throwing as getters

Calling a setter often implicitly involves calling a getter, so it may 
make sense to require the setter to be at least as throwing as the 
getter. Absent feedback to this effect from implementors, however, my 
instinct is to leave them independent, as there may be use cases where 
a get can throw but a set cannot. (For instance, if an instance faults 
in data from an external source on demand, but only writes that data 
back when a `save()` call is invoked, `get` may throw if it's unable to 
fault in the data, but `set` would never need to throw.)

### Make `rethrows` setters throwing if `newValue` is throwing

`newValue` is sort of like a parameter to the setter, so it might 
technically be more consistent for `rethrows` to consider `newValue` 
when deciding if a particular invocation `throws` or not. However, I 
can't imagine a case where this would be appropriate behavior, and 
considering only the subscript parameters makes the getter and setter 
work better together.

### Don't include `willSet throws`

The use of `willSet throws` to make a stored property throwing is a bit
funky and could be omitted. I decided to include it because, if it does 
not exist, people will fake it with private properties anyway.

### Include `didSet throws`

There is no technical reason not to support `didSet throws` accessors, 
which would allow stored properties to be made throwing. However, this 
would usually be the wrong thing to do because it would leave the 
errant value in the property. If compelling use cases for it were 
cited, however, `didSet throws` could be added.

### Permit `try` on `&foo` itself, rather than the call using it

As specified, if `foo` has a throwing accessor and you want to pass it 
to a function `bar` with an inout parameter, you have to write this:

	try bar(&foo)

In theory, we could instead allow you to mark only the `&` operator, 
leaving the rest of the expression uncovered by the `try`:

	bar(try &foo)

This would make the source of the potential error more obvious, but it 
might make the semantics less clear, because `try &foo` can throw 
*after* the call is finished in addition to *before*. I judge the 
latter issue to be more serious.

### Try to convert keyed getter/setter methods to subscripts

Swift could conceivably apply heuristics to discover Objective-C method 
pairs that can be expressed as subscripts. For instance, the `NSURL` 
method pair cited in the Motivation section:

```swift
func getResourceValue(_ value: AutoreleasingUnsafeMutablePointer<AnyObject?>, forKey key: String) throws
func setResourceValue(_ value: AnyObject?, forKey key: String) throws
```

Could be imported like this:

```swift
subscript (resourceValueFor key: String) -> AnyObject? {
	get throws
	set throws
}
```

There are several reasons not to do this:

* There is no established pattern for throwing subscripts in 
  Objective-C, so any we might establish would be mistake-prone.
* [SE-0044](https://github.com/apple/swift-evolution/blob/master/proposals/0044-import-as-member.md)
  does not currently include subscripts, so there is no proposal 
  pending which would allow the heuristic to be tweaked or the 
  import logic to be invoked manually. (This is arguably an oversight 
  in SE-0044.)
* Many such cases would benefit from a human looking at the design. In 
  the `NSURL` case, for instance, a human looking at the broader type 
  might prefer a design like this:

```swift
var resourceValues: ResourceValues { get } 

struct ResourceValues {
	subscript (key: String) -> AnyObject? {
		get throws { ... }
		set throws { ... }
	}
	
	func get(for keys: [String]) throws -> [String: AnyObject] { ... }
	func set(from dict: [String: AnyObject]) throws { ... }
	
	func removeCachedKeys() { ... }
	func removeCachedKey(key: String) { ... }
	func setTemporaryValue(_ value: AnyObject?, for key: String) { ... }
}
```

### Automatically export throwing subscript accessors to Objective-C

Throwing subscript accessors can only be exported by specifying a name 
using an `@objc` property. It might be nice to export them by default,
but Objective-C doesn't have an established pattern for throwing 
subscript accessors, so it's not clear how these methods would be 
named.

### Add a `nothrows` keyword

Leaving an accessor's throwing behavior unspecified could make it 
automatically take on the behavior required by the type's superclass or 
conformed protocols. However, this would require a way to explicitly 
state that an accessor could *not* throw, along the lines of the 
rarely-used but necessary `nonmutating` keyword.

I have chosen not to do this because Swift generally does not allow you 
to infer parts of a member's signature, and because I cannot come up 
with a way to spell this keyword that isn't ugly as sin.
