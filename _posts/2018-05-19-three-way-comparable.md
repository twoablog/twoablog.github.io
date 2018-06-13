---
layout: post
title: The "three-way comparable" protocol
date: 2018-06-12
excerpt: "Efficient, automagically generated comparison operators in swift."
tags: [Swift, C++]
---

Swift's __protocols__ and their __extensions__ make it one of the most developer-friendly languages in the world. One great example is the `Comparable` protocol. By simply implementing the `==` and `<` operators for your custom types, conformance with `Comparable` gives you the operators `!=`, `>`, `<=` and `>=` for free. This way you can make your intent clearer when you write code, using statements like `if x <= y { ... }` instead of `if x < y || x == y { ... }`, or even worse (but more efficient) `if !(y < x) { ... }`.

The default implementations you get are also very efficient. Each one of the operators `>`, `<=` and `>=` will call your custom implementation of `<` exactly once, and the operator `!=` will call your custom implementation of `==` exactly once. Not bad. Add to that the fact that swift 4.1 (and later) can automatically synthesise the `==` operator if you declare conformance with `Equatable`, and your programming comfort is optimised. Great, see you next week. Unless...

...you want to react differently to each case `x < y`, `x > y` and `x == y` and your comparison operation is very expensive. In other words, suppose you have a comparable type `MyType` and somewhere in your code you use it as follows:

```swift
let x = MyType(...)
let y = MyType(...)

if x < y {
    print("The smaller one is x!")
} else if x > y {
    print("The smaller one is y!")
} else {
    print("It's a tie!")
}
```

If comparing instances of `MyType` costs a lot of time or other resources, you should be very annoyed at the necessity to compare `x` and `y` twice.

In these troubled times we turn to our ancestors for their wisdom... and remember `strcmp`!

For decades, C-programmers have been using the function `strcmp(char const *, char const *)` to compare (null-terminated) strings with respect to their alphabetical order. The return type of this function is `int`, and the contract is as follows: If the first string is __less__ than the second (in terms of alphabetical order), the returned integer will be __less__ than zero. If it is __greater__ than the second string, the integer will be __greater__ than zero. And if the strings are __equal__, the returned value will be __equal__ to zero. The magnitude of the returned integer is not specified by the standard and is implementation dependent.

I know, I know: Returning an integer to convey an ordering is sooo 1989. But the concept is still brilliant, and now we have the tools to make it all elegant and swifty. We'll use a beautiful custom operator:

```swift
infix operator <=>: ComparisonPrecedence
```

This operator is sometimes called the *spaceship*-operator (perhaps a [TIE interceptor](https://www.starwars.com/databank/tie-interceptor)?) and is the same one recently added to the C++20 draft, also for the purpose of three-way comparison, following a proposal by Herb Sutter. Now we need something better than `Int` to return from our comparisons:

```swift
enum Order {
	case increasing
	case equal
	case decreasing
}
```

Finally, we can write a protocol for types that can be three-way compared:

```swift
protocol ThreeWayComparable {
	static func <=> (lhs: Self, rhs: Self) -> Order
}
```

We'll get to the performance benefits in a minute, but first let us talk about conforming to both `Comparable` and `ThreeWayComparable` simultaneously, so that we can use all of the operators `==`, `!=`, `<`, `<=`, `>`, `>=` and `<=>`. We can provide default implementations for each protocol using the other one as follows:

```swift
extension Comparable where Self: ThreeWayComparable {
	static func == (lhs: Self, rhs: Self) -> Bool {
		return (lhs <=> rhs) == .equal
	}

	static func < (lhs: Self, rhs: Self) -> Bool {
		return (lhs <=> rhs) == .increasing
	}
}

extension ThreeWayComparable where Self: Comparable {
	static func <=> (lhs: Self, rhs: Self) -> Order {
		if lhs < rhs {
			return .increasing
		} else if lhs == rhs {
			return .equal
		} else {
			return .decreasing
		}
	}
}
```

Of course, for types for which we explicitly implement both protocols, our explicit implementations will be used and not these default ones. Notice that we do not provide a default implementation for `>` using `ThreeWayComparable` for example. This would likely cause some headache since `Comparable` already provides default implementations for these other operators.

We are ready for some performance testing. We could use an example in which we compare long arrays of values in terms of lexicographical order (i.e. basically use our operator to re-implement `strcmp`) but this is not where the true strength of the protocol lies. Such examples with a long sequence of elements, but where the element type is cheap to compare, would be very easy to solve without the protocol. The really interesting cases are those where the cost of comparison comes not only from the sheer size of the data, but from the complexity of its structure as well. In other words we want types that have members of other types that have members of other types and so on and so forth, achieving a hierarchy with great depth as well as width.

For our performance tests we'll use the type `ComparePair` defined below:

```swift
struct Pair<Value: Comparable & ThreeWayComparable> {
	var a: Value
	var b: Value
}

extension Pair: Comparable {
	static func == (lhs: Pair<Value>, rhs: Pair<Value>) -> Bool {
		return lhs.a == rhs.a && lhs.b == rhs.b
	}

	static func < (lhs: Pair<Value>, rhs: Pair<Value>) -> Bool {
		return (lhs.a < rhs.a) || (lhs.a == rhs.a && lhs.b < rhs.b)
	}
}

extension Pair: ThreeWayComparable {
	static func <=> (lhs: Pair<Value>, rhs: Pair<Value>) -> Order {
		let aOrder = lhs.a <=> rhs.a
		return (aOrder == .equal) ? (lhs.b <=> rhs.b) : aOrder
	}
}

extension Int: ThreeWayComparable {}

// Depth-14 nesting means we're holding 2^14 = 16384 `Int`'s in each
// `ComparePair` instance. That's 128 KB on a 64-Bit machine.
typealias ComparePair =
	Pair<
		Pair<
			Pair<
				Pair<
					Pair<
						Pair<
							Pair<
								Pair<
									Pair<
										Pair<
											Pair<
												Pair<
													Pair<
														Pair<
															Int>>>>>>>>>>>>>>
```

My machine started to give up at depth 15 (i.e. 15 nested `Pair`'s) and either didn't manage to compile the code or crashed with a "bad access" exception when running. We still need a reasonable way of initialising `ComparePair`.

```swift
protocol DefaultInitializable {
	init()
}

extension Pair: DefaultInitializable where Value: DefaultInitializable {
	init() {
		self.init(a: Value(), b: Value())
	}
}

extension Int: DefaultInitializable {}
```

Finally, it's time run some tests. I'm not terribly concerned with precise measurements here, but rather with showing that the general idea of the `ThreeWayComparable` protocol has some performance advantages in specific situations. I added the following test case to my xcode project:

```swift
class ThreeWayComparableTests: XCTestCase {
	func test() {
		let lhs = ComparePair()
		var rhs = lhs
		rhs.b.b.b.b.b.b.b.b.b.b.b.b.b.b -= 1

		var order: Order? = nil

		measure {
			if lhs < rhs {
				order = .increasing
			} else if lhs > rhs {
				order = .decreasing
			} else {
				order = .equal
			}
		}

		XCTAssertEqual(order, .decreasing)
	}
}
```

We'll call this the _decreasing_ case. We're also interested in the _increasing_ case, where we replace the `-= 1` with `+= 1` (and fix the final `XCTAssertEqual`) and the _equal_ case, where we do not modify `rhs` at all (again, fixing the final assertion). Moreover, in each one of these three cases we want to examine the effects of replacing the _traditional comparison_ in the `measure`-block with a _three-way comparison_:

```swift
		measure {
			order = lhs <=> rhs
		}
```
Using the _traditional comparison_, the runtime is strongly dependent on which of the three cases we're testing. In the _decreasing_ and the _equal_ cases, our first check (`lhs < rhs`) is false, so we have to perform a second comparison. As expected, the runtime is about 2x longer than in the _increasing_ case. On my machine the results were about 0.014 seconds on average in the _decreasing_ and _equal_ cases, against about 0.007 seconds on average in the _increasing_ case. All tests had very stable measurements.

Using _three-way comparison_, the runtime in all three cases is essentially the same. On my machine the tests resulted in about 0.002 seconds on average.

A word of caution: I believe the "achievement" here is the fact that the `<=>` operation is __not slower__ than the `<` operation, and has the advantage that we never have to perform a second comparison to find out exactly how two instances are ordered. The fact that a single `<=>` operation was also almost 4x faster than a single `<` operation could be specific to this example. The performance is affected by the machine, compiler optimisations and implementation details of the language, and in different situations the speed-up might be smaller, or even non-existent, as the next example shows.

I ran similar tests simply comparing arrays of 16384 `Int`'s (using the lexicographical order). For the _traditional comparison_ I used the method `Array.lexicographicallyPrecedes(...)`, from the standard library, while the `<=>` operator was implemented as below:

```swift
extension Array: ThreeWayComparable where Element: ThreeWayComparable {
	public static func <=> (lhs: Array<Element>, rhs: Array<Element>) -> Order {
		for index in lhs.indices {
			if index == rhs.count {
				return .decreasing
			} else {
				let order = lhs[index] <=> rhs[index]

				if order != .equal {
					return order
				}
			}
		}

		return (lhs.count < rhs.count) ? .increasing : .equal
	}
}
```

In this case, each call of `lexicographicallyPrecedes` took approximately 0.005 seconds, while calling the `<=>` operator took about 0.004 seconds on average, which is a much more modest speed-up, and is absolutely insignificant because my tests were not designed to measure with such precision. But again, in the worst-case scenario the runtime doubled for _traditional comparison_ but remained unchanged for _three-way comparison_, justifying the use of this protocol.

---

Of course, the intent of using the new protocol was mainly to improve performance *on average* by never having to compare the same pair of instances twice. Nevertheless, the consistency of the runtime is also a very attractive quality in the field of cryptography, where so-called *timing attacks* attempt to use the amount of time a program takes to perform some computation, to figure out information about its input or its output or some intermediate value, when these should be kept secret.

## To summarise...

By implementing  the `ThreeWayComparable` protocol for your custom types, which requires you to implement a single binary operator, you get a fast and runtime-consistent way of finding out whether a pair of instances is in increasing or decreasing order, or whether the two instances are equal, all in a single operation. If you then simply declare your custom type to conform to `Comparable`, you get that conformance (and all the neat operators that come with it) for free. Oh, and we should never underestimate how much we can learn from older, battle-tested programming languages.
