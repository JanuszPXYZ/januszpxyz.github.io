---
layout: post
title: "From `Any` to Type Safety: Revisiting Sentry's Metrics API"
date: 2026-03-22 18:53:00 +0100
categories: Software
description: "How deep does Swift's type inference actually go? Digging into Sentry's Metrics API reveals a compiler that's more capable than it's given credit for."
---
After a long period of inactivity, I am coming back with a new blog post. Apart from being bogged down with couple of personal projects that I really wanted to finish, I didn’t have time to explore any interesting topics that I wanted to write about, but that changed after I attended Mobileheads meetup hosted by Sentry here in Vienna. It was a very interesting evening filled with great talks from developers working at the company. Stefan Pölz’s presentation about observability in .NET Maui gave me insight into C# and some of the challenges he’s facing when working with both the language and the framework on a day to day basis. But, given my affinity for Swift, the talk that inspired me to write this blog post was [Phil Niedertscheider](https://sentry.engineering/about/philniedertscheider)’s presentation about type safety in Swift and how he’s utilizing the power of Protocol Oriented Programming in the new Metrics feature in Sentry Cocoa SDK. Before I dive deep into the topic and present some findings, two important things:

1. I will focus only on one section of Phil’s presentation, namely dealing with heterogeneous arrays in Swift and the challenges that this particular issue poses. If you’re interested in the topic, then I cannot recommend enough his blog posts: https://sentry.engineering/blog/building-type-safe-metrics-api-in-swift-part-i and part two: https://sentry.engineering/blog/building-type-safe-metrics-api-in-swift-part-ii
2. I may be completely wrong about everything that I write here and this whole blog post could be labeled as “talking out of my ass”, but I am more than happy to be proven wrong as it’s the only way to learn.

Having these two things out of the way, let’s proceed and briefly describe what is the issue we’re dealing with here.

## Heterogeneous Arrays and The Problem with **`Any`**

By definition, Arrays in Swift are homogeneous. **`[T]`** means every element is the same type **`T`**. This isn’t a limitation so much as a **guarantee:** compiler knows what it’s working with. However, in the case that Sentry is dealing with here, this assumption is quickly broken by the attribute values. A single metric event might carry a string endpoint, an integer count, or a boolean flag. All are valid, but different types. The naive solution is to use **`Any`**, Swift’s type-erased “escape type”. It compiles, runs, and accepts everything you throw at it. But it comes at a cost, as Phil puts it: “*One major drawback of using **`Any`** as the value of our attributes is missing compile-time hints if the passed-in value is not one of our supported attribute value types.”* 

The solution to the **`Any`** headache in Sentry’s case was to define an enum type with associated values, **`SentryAttributeContent`**, which looks like this:

```swift
enum SentryAttributeContent {
    case string(String)
    case boolean(Bool)
    case integer(Int)
    case double(Double)
    case stringArray([String])
    case booleanArray([Bool])
    case integerArray([Int])
    case doubleArray([Double])
}
```

This is Swift’s equivalent of a union type. Instead of accepting anything and hoping for the best at runtime, the compiler now enforces that attribute values are one of these known cases. The call site gets cleaner too, as we’re no more wrapping values manually, just use the enum case syntax:

```swift
protocol SentryMetricsApiProtocol {
    func count(key: String, value: UInt, attributes: [String: SentryAttributeContent])
}

SentrySDK.metrics.count(key: "checkout.completed", value: 1, attributes: [
    "payment_method": .string("apple_pay"),
    "cart_items": .integer(3),
    "total_amount": .double(99.99)
])
```

The problem is that this **`enum`** is closed. Every type that you want to support must be explicitly declared as a case, and if you want to pass a custom type like **`User`**, for example, you’re out of luck unless you extend the enum yourself. 

The cleaner solution is to invert the relationship: instead of the SDK “owning” all possible types in a closed enum, define a protocol that any type can conform to by declaring how it maps to **`SentryAttributeContent`**:

```swift
protocol SentryAttributeValue {
    var asSentryAttributeContent: SentryAttributeContent { get }
}
```

Now having this protocol requires us to update the **`SentryMetricsApiProtocol`** API, like so:

```swift
protocol SentryMetricsApiProtocol {
    func count(key: String, value: UInt, attributes: [String: any SentryAttributeValue])
}
```

**`count`** function signature uses a boxed protocol type, which accepts any conforming type as a value in the **`attributes`** dictionary. Having all of this in place, we can now leverage Swift’s extensions and extend the types that we need to conform to **`SentryAttributeValue`**. Great, but you may be asking how all of this connects to the arrays that started this whole discussion. And now we get to the part that is very interesting.

## *Encountering Compiler Limitations*

In this section of the Sentry’s blog post, Phil discusses the need to extend Swift’s **`Array`** to conform to the **`SentryAttributeValue`**, but “*only if the array contains elements which are one of our supported types.”* Turns out that having multiple multiple conformances of the same protocol narrowed down to specific element types is unsupported, so this:

```swift
extension Array: SentryAttributeValue where Element == Int {
    var asSentryAttributeContent: SentryAttributeContent {
        .integerArray(self)
    }
}

extension Array: SentryAttributeValue where Element == Double {
    var asSentryAttributeContent: SentryAttributeContent {
        .doubleArray(self)
    }
} 
```

will result in Swift compiler complaining. What can be done here is to use a generic **`where`** clause with the **`SentryAttributeValue`** protocol introduced earlier. Here’s how it looks like:

```swift
extension Array: SentryAttributeValue where Element == SentryAttributeValue {
    var asSentryAttributeContent: SentryAttributeContent {
        if Element.self == Bool.self, let values = self as? [Bool] {
            return .booleanArray(values)
        }
        // ... and other cases

        // Fallback to converting to strings
        return .stringArray(self.map { element in
            String(describing: element)
        })
    }
}
```

Types conforming to the **`SentryAttributeValue`** can now be passed in arrays:

```swift
SentrySDK.metrics.count(
    key: "order.placed",
    attributes: [
        "customer_id": "cust_456",           // ✅ String works
        "product_ids": ["sku_1", "sku_2"],   // ✅ Array of String works
        "quantities": [2, 1, 3]              // ✅ Array of Integer works too
    ]
)
```

But as you can see the two arrays that have been passed are homogeneous, which **conform to the `SentryAttributeValue`** protocol. The real issue is “*mixing multiple types adopting* **`SentryAttributeValue`** *into a single array*”. And now we get to the part where I started experimenting with the code just to see what’s happening “under the hood”. Phil notes the following: “*We hoped that the compiler would somehow be smart enough to understand that now it’s an array of **`SentryAttributeValue`**, but instead it fell back to an array of **`Any`***.”

This part piqued my interest, because I always assumed that Swift compiler would, in fact, preserve type information, especially with the aid of the generic **`where`** clause. Here’s the code snippet posted on [sentry.engineering](http://sentry.engineering) blog:

```swift
struct ProductID: SentryAttributeValue {
    var asSentryAttributeContent: SentryAttributeContent {
        return .string("product_1")
    }
}

struct CategoryID: SentryAttributeValue {
    var asSentryAttributeContent: SentryAttributeContent {
        return .string("electronics")
    }
}

SentrySDK.metrics.count(
    key: "page.viewed",
    attributes: [
        // Mixed array of types adopting SentryAttributeValue
        // Both return string content, so this could be a string[]
        // ❌ Compiler sees [Any], not [SentryAttributeValue], and fails
        "related_items": [ProductID(), CategoryID()]
    ]
)
```

I cloned the [sentry-cocoa](https://github.com/getsentry/sentry-cocoa) repo from GitHub, found the relevant code and started digging. Here are my findings:

### 1. `where` clause on the `Array` extension preserves type information

But only when the compiler **has enough context to infer that `Element == any SentryAttributeValue`**. That context can come from two places:

- **Function signature propagation** (more about that later)
- **Explicit type annotation - `let sampleArray: [any SentryAttributeValue] = [ProductID(), CategoryID()]`**

Without either of those, the compiler has nothing to aim for and falls back to **`[Any]`** and throws the “*Heterogeneous collection literal could only be inferred to* **`[Any]`**; *add explicit type annotation if this is intentional*” error.

What’s fascinating here is the fact that the **`where`** clause and type context propagation seem to be “complementing mechanisms”. The **`where`** clause alone isn’t sufficient, and type context propagation alone isn’t sufficient. It’s the combination of the two that produces the desired behavior.

### 2. Type context propagates inward from the function signature

When I cloned the sentry-cocoa repo, the **`Array`** extension defined in the **`SentryAttributeValue`** looked like this:

```swift
// https://github.com/getsentry/sentry-cocoa/blob/main/Sources/Swift/Protocol/SentryAttributeValue.swift
extension Array: SentryAttributeValue {
    /// Converts an array to a SentryAttributeContent value.
    ///
    /// This extension cannot be scoped to `where Element == SentryAttributeValue` because:
    /// - Mixed arrays (arrays containing elements of different types) are automatically converted to `[Any]` by Swift
    /// - If we used `where Element == SentryAttributeValue`, mixed arrays would not compile, preventing users
    ///   from passing heterogeneous arrays
    /// - We accept this trade-off: while we can't enforce compile-time safety for mixed arrays, we can convert
    ///   the values to strings at runtime without losing any data
    ///
    /// Arrays can be heterogenous, therefore we must validate if all elements are of the same type.
    /// We must assert the element type too, because due to Objective-C bridging we noticed invalid conversions
    /// of empty String Arrays to Bool Arrays.
    public var asSentryAttributeContent: SentryAttributeContent {
        if Element.self == Bool.self, let values = self as? [Bool] {
            return .booleanArray(values)
        }
        if Element.self == Double.self, let values = self as? [Double] {
            return .doubleArray(values)
        }
        if Element.self == Float.self, let values = self as? [Float] {
            return .doubleArray(values.map(Double.init))
        }
        if Element.self == Int.self, let values = self as? [Int] {
            return .integerArray(values)
        }
        if Element.self == String.self, let values = self as? [String] {
            return .stringArray(values)
        }
        if let values = self as? [SentryAttributeValue] {
            return castArrayToAttributeContent(values: values)
        }
        return .stringArray(self.map { element in
            String(describing: element)
        })
    }
    
    // *rest of the code
    // ...*
}
```

As you can see there is no generic **`where`** clause defined on that extension. So, what I did was bring back the **`where`** clause, modify the **`asSentryAttributeContent`** a bit so that the compiler would not complain, and see what kind of output I’d get when running the example given in the blog post. 

The results of my tiny experiment made me extremely puzzled, because printing the type of the values in the **`attributes`** dictionary, I got this: **`Array<SentryAttributeValue>`.** Wait, that can’t be right, shouldn’t it be **`Array<Any>`**? Initially I thought I made a mistake somewhere, or that the **`type(of:)`** method does something at runtime and already has the necessary type information. In order to verify what actually happens I decided to see what Swift compiler does with the code, as this is the most authoritative source on the matter. I emitted the SIL output to the terminal and looked for some clues that would help me out here. And there it was:

```nasm
  %34 = function_ref @$ss27_allocateUninitializedArrayySayxG_BptBwlF : $@convention(thin) <τ_0_0> (Builtin.Word) -> (@owned Array<τ_0_0>, Builtin.RawPointer) // user: %35
  %35 = apply %34<any SentryAttributeValue>(%33) : $@convention(thin) <τ_0_0> (Builtin.Word) -> (@owned Array<τ_0_0>, Builtin.RawPointer) // users: %37, %36
```

What this bit is showing is the compiler calling the **`_allocateUninitializedArray`**, an internal Swift function that allocates new array. The critical part is the generic parameter it is being instantiated with: **`<any** **SentryAttributeValue>**`. This means the compiler decided on **`any** **SentryAttributeValue**` as the element type **at the moment of allocation**, before any elements are inserted, or any runtime casting happens. Looking more into the SIL output, immediately after the compiler decided on the element type, there is this:

```nasm
  %36 = tuple_extract %35 : $(Array<any SentryAttributeValue>, Builtin.RawPointer), 0 // users: %38, %53
  %37 = tuple_extract %35 : $(Array<any SentryAttributeValue>, Builtin.RawPointer), 1 // user: %38
  %38 = mark_dependence %37 : $Builtin.RawPointer on %36 : $Array<any SentryAttributeValue> // user: %39
  %39 = pointer_to_address %38 : $Builtin.RawPointer to [strict] $*any SentryAttributeValue // users: %46, %43
```

The array’s internal buffer is typed as **`$*any SentryAttributeValue`**, which is a pointer to existential values.

Great, but what does this have to do with type context propagation? Quite a lot actually! The source code at this point is just this:

```nasm
"related_items": [ProductID(), CategoryID()]
```

There is no explicit type annotation **anywhere** near that array literal. The only source of **`any** **SentryAttributeValue**` type information in scope is the **`count`** function signature:

```swift
func count(key: String, value: UInt, attributes: [String: any SentryAttributeValue])
```

Therefore, the compiler must have worked **backwards** from that signature through the dictionary literal and into the array literal to arrive at our desired existential type. SIL output proves it happened at compile time, not as a runtime coercion. For comparison, check the SIL output when I used the **`Array`** extension as it is defined in sentry-cocoa codebase (without the **`where`** clause):

```nasm
  %54 = function_ref @$ss27_allocateUninitializedArrayySayxG_BptBwlF : $@convention(thin) <τ_0_0> (Builtin.Word) -> (@owned Array<τ_0_0>, Builtin.RawPointer) // user: %55
  %55 = apply %54<Any>(%53) : $@convention(thin) <τ_0_0> (Builtin.Word) -> (@owned Array<τ_0_0>, Builtin.RawPointer) // users: %57, %56
  %56 = tuple_extract %55 : $(Array<Any>, Builtin.RawPointer), 0 // users: %58, %69
  %57 = tuple_extract %55 : $(Array<Any>, Builtin.RawPointer), 1 // user: %58
  %58 = mark_dependence %57 : $Builtin.RawPointer on %56 : $Array<Any> // user: %59
  %59 = pointer_to_address %58 : $Builtin.RawPointer to [strict] $*Any // user: %66

```

Same allocation function, different generic parameter. The compiler simply gave up on finding a meaningful type and “defaulted” to **`Any`**, even though the function signature is identical! This contrast shows that there is evidence that the **`where`** clause and type context propagation are cooperating.

### 3. The `where` clause enforces compile-time type safety

A simple test to check if type safety is working is to do something like this:

```swift
struct NotAnAttributeValue {} // No SentryAttributeValue conformance

SentrySDK.metrics.count(
    key: "test",
    value: 1,
    attributes: [
        "should_fail": [NotAnAttributeValue(), NotAnAttributeValue()]
    ]
)
```

With the **`where`** clause the compiler is going to immediately complain with the following error: **`Cannot convert value of type '[NotAnAttributeValue]' to expected dictionary value type 'any SentryAttributeValue’`**. Without the **`where`** clause the code is going to compile silently and fall back to string. Of course, we had the confirmation about the compile-time type safety earlier when we checked the SIL output, but nonetheless it never hurts to perform a simple test like the one above.

### 4. The `String` ”fallback” for heterogeneous conforming types is correct behavior

In the sentry-cocoa codebase, the **`castArrayToAttributeContent`** looks like this:

```swift
    private func castArrayToAttributeContent(values: [SentryAttributeValue]) -> SentryAttributeContent {
        // Empty arrays cannot determine the intended type, so default to stringArray as a safe fallback
        guard !values.isEmpty else {
            return .stringArray([])
        }
        
        // Check if the values are homogeneous and can be casted to a specific array type.
        // Each cast method optimizes by checking the first element first, avoiding full iterations
        // when the first element doesn't match the expected type.
        if let booleanArray = castValuesToBooleanArray(values) {
            return booleanArray
        }
        if let doubleArray = castValuesToDoubleArray(values) {
            return doubleArray
        }
        if let integerArray = castValuesToIntegerArray(values) {
            return integerArray
        }
        if let stringArray = castValuesToStringArray(values) {
            return stringArray
        }
        
        // If the values are not homogenous we resolve the individual values to strings
        return .stringArray(values.map { value in
            switch value.asSentryAttributeContent {
            case .boolean(let value):
                return String(describing: value)
            case .double(let value):
                return String(describing: value)
            case .integer(let value):
                return String(describing: value)
            case .string(let value):
                return value
            default:
                return String(describing: value)
            }
        })
    }
```

When we run the example with the **`[ProductID(), CategoryID()]`** array, we can see that the path being picked is the **`if let stringArray = castValuesToStringArray(values) { return stringArray }`** branch of the **`castArrayToAttributeContent`**. We can add a third object to the mix, defined like this:

```swift
struct TypeID: SentryAttributeValue {
    var asSentryAttributeContent: SentryAttributeContent {
        return .integer(12)
    }
}
```

and plugging it into the array with **`ProductID`** and **`CategoryID`** will result in the individual values being resolved to strings, i.e., the final option in the **`castArrayToAttributeContent`** method.

### 5. Removing the `where` clause silently undermines type safety

What started as a close reading of a compiler error turned into something more interesting: a demonstration of Swift’s type inference capabilities. Given sufficient type context (whether from an explicit annotation or a function signature) the compiler is remarkably precise about preserving the existential type information all the way down through nested literals, as was unambiguously proven by the SIL output. My little experiment also revealed where Swift’s “ceiling” is. The **`where`** clause and type context propagation have to cooperate perfectly for this to work: remove either one and the whole chain breaks. 

Throughout this exploration, there was this one thought that kept resurfacing: how awesome would it be if Swift had Zig’s **`comptime`** feature. It’s not a criticism of Swift by any means, but a loose observation about what becomes possible when the compile-time/runtime boundary is a first-class language concern rather than something you have to “navigate around”. Swift is of course still evolving (although whether or not its evolution is taking a positive turn is up for discussion), but **`comptime`** remains a glimpse of a different way of thinking about the problem entirely.

I thoroughly enjoyed this deep dive into Swift and Sentry API. Huge thanks (and a special shoutout) to Sentry for hosting the Mobileheads meetup, to Stefan Pölz for an interesting talk and to [Phil Niedertscheider](https://sentry.engineering/about/philniedertscheider) for inspiring me to write this article. As I said in the beginning, I may be totally wrong on this and the removal of the **`where`** clause from the **`Array`** extension may be the right call for Sentry’s Metrics API. I acknowledge that I may be missing some foundational building blocks, but as always, I am more than open to learning and what better way to learn than through making mistakes!

Janusz
