# `𝑓(x)` FunctionalAPI 

Some collection of functions and types that are useful in day to day development.

---
# Functions 

# `map`

To help a bit with composability some existing map functions defined on `Array`, `Optional` and `Result` are made as global symbols.

```swift
public func map<A,B>(_ a: A?, _ f: (A) -> B) -> B? { ... 

public func map<A,B>(_ a: [A], _ f: (A) -> B) -> [B] { ... 

public func map<A,B>(_ a: Result<A,Error> , _ f: (A) -> B) -> Result<B, Error> { ...
```
 
## `identity`
Whole implementation looks like this:

```swift
func identity<A>(_ a: A) -> A { a }
```

This might seem as a useless function but every time you write in your code `{ $0 }` you are writing this exact same function. Let's see some example:

```swift
[1,2,3].map{ $0 } // [1,2,3]
```

Same cane be done with identity but with extra clarity:

```swift
[1,2,3].map( identity ) // [1,2,3]
```
## `product`

If you are thinking _cartesian product_ then you are right! Just take up to 5 arrays of stuff and you will get an array of tuples with all combinations.

Let's see a easy example, for more complicated check out tests:

```swift
let aa = [1,2]
let bb = ["a", "b"]

product(aa, bb)

// (1, "a")
// (1, "b")
// (2, "a")
// (2, "b")
```

### Where this function is useful?

It depends what you need. I use it to generate variations in my snapshot tests. One array contains device, other size class, language direction... you get the idea ;)

---
# Types

# `Either`

Either is one of those types that you already know in Swift but it's specialized in some values. 

Here is it:

```swift
enum Either<Left, Right> {
    case left(Left)
    case right(Right)
}
```

Slo it's an enum with two cases. That should ring at least two bells. Usually the `right` one is associated with the `good` or `success` or `dextra`. The `left` on the other hand is for `bad`, `failure`, `error` or you can say `sinistra`.

At least two types that are common in Swift are a specialization of this `Either`. 

You can take a look at _Optional_ and match `left` with _none_ case. Also you can take a look at _Result_ and match if _failure_ with a `left` case.

So here you have a more generic type that you can use. It's useful not only when you have to validate stuff. But also when there is a need to return different types.

## Computed Properties

To make life a bit easier there are defined some helper properties. 

Handy properties for logic checks. 

```swift
var isLeft : Bool // true if `left` case
var isRight: Bool // true if `right` case
```

When you want to get to a value but you can deal with optional.

```swift
var right: Right?
var left : Left?
```

Take a look at [OptionalAPI](https://github.com/sloik/OptionalAPI) to see how working with Optional can be a pleasure.

## `map`s

```swift
func map<R>(_ transform: (Right) -> R) -> Either<Left,R>
```

It's a function that takes a function expecting an instance of `Right` and produces a new instance of `Either<Left, R>`. Type `R` means `NewRight` but is abbreviated.

Either treats it's right value a bit specially. This is analogous how Result and Optional treat some of their own cases. 

That means that if you want to `map` an `Either` then by default you will be given an instance of a `Right` type.

So if you have:
```swift
func increment(_ i: Int) -> Int { i + 1 }

let right = Either<String,Int>.right(42)

right
    .map(increment) // .right(43)
```

Final result is a new instance of `Either<String,Int>` with it's `right` case holding `43`. Same thing with _left_ would do nothing.

```swift
let left = Either<String,Int>.left("I'm left")

left
    .map(increment) //  .left("I'm left")
```

### `rightMap`

```swift
func mapRight<R>(_ transform: (Right) -> R) -> Either<Left, R>
```

You can be explicit about it and call `rightMap` that is just a wrapper around `map`.

### `leftMap`

```swift
func mapLeft<L>(_ transform: (Left) -> L) -> Either<L, Right>
```

However if you want to transform a `left` value you can use `leftMap`.  Same story as for right but for left.

```swift
left
    .map({ $0.uppercased() }) // .left("I'M LEFT")
```

### `biMap`

```swift
func mapBi<L,R>( _  leftTransform:  (Left) -> L,
                 _ rightTransform: (Right) -> R)
     -> Either<L, R>
```

This function takes two other functions as arguments. Left is the _left transform_ and the right is the _right transform_. This results in a new instance of `Either<L,R>`.

This one is a `map` that combines `leftMap` and `rightMap`. More common name for it is `biMap` but I would like it be handy in the IDE and you can start typing just _map_ and see what's there.

This `biMap` can be used when you want to combine those transformations in to one statement:

```swift
right
    .mapBi({ $0.uppercased() }, increment) // .right(43)

left
    .mapBi({ $0.uppercased() }, increment) // .left("I'M LEFT"
```

## `flatMaps`

Same as map above, but with one difference. 

```swift
func flatMap<R>(
    _ transform: (Right) -> Either<Left, R> 
    ) -> Either<Left, R>

func flatMapLeft<L>(
    _ transform: (Left) -> Either<L, Right>
    ) -> Either<L, Right>
```

This time a transform function returns another Either. This wold result in an **Either Either** like `Either<Left, Either<Left, R>>`. So using this flatMap (or another name you can fins `bind`) you can remove one layer of nesting.

Just as a side note, whenever you are using `flatMap` you are doing the dreaded **monadic computation**. You were calling `flatMap` on optionals and it was fine. And yes Optional is a `Monad`. If you want you can Google it just don't overthink it and you will be just fine 😎

# Utils

## `either`

```swift
func either<L,R,T>(
    _  leftTransform: @escaping (L) -> T,
    _ rightTransform: @escaping (R) -> T) -> (Either<L,R>) -> T
```

This might not be so common, but it's a function that returns another function. The way it works you start with providing two functions. Left transform knows kow to produce `T` given an `L`. Right transform knows kow to produce `T` given an `R`. 

Next step is the returned function. This one expects and `Either<L,R>` and when you provide an instance of this either then it will produce and instance of `T`. 

## `lefts`

```swift
func lefts<L,R>(_ eithers: [Either<L,R>] ) -> [L]
```

I guess type says it all. Give this function an array of either-s and in return you will get an array of `L`s.

## `rights`

```swift
func rights<L,R>(_ eithers: [Either<L,R>] ) -> [R]
```

I guess type says it all. Give this function an array of either-s and in return you will get an array of `R`s.

## `partitionEithers`

Returns a tuple containing all `L` values in the `left`/first array. And all the `R` in the `right`/ second array.

```swift
let eithers: [Either<String,Int>] = 
    [ .left("A"), 
      .right(42), 
      .left("B"), 
      .right(24) ]

let (lefts, rights) = partitionEithers(eithers)

lefts  // ["A", "B"]
rights // [42, 24]
```


# Free Functions

In this package there are defined global symbols or functions to all of those types. It tuns out that after some curring they can compose better better. Not yet thou... but you can take a look at this great library for composing functions called [Overture](https://github.com/pointfreeco/swift-overture).


---

T.B.C.
