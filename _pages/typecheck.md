---
permalink: /typecheck
title: Type checking
toc: true
---

Luau supports a gradual type system through the use of type annotations and type inference.

## Type inference modes

There are three modes currently available. They must be annotated on the top few lines among the comments.

* `--!nocheck`,
* `--!nonstrict` (default), and
* `--!strict`

`nocheck` mode will simply not start the type inference engine whatsoever.

As for the other two, they are largely similar but with one important difference: in nonstrict mode, we infer `any` for most of the types if we couldn't figure it out early enough. This means that given this snippet:

```lua
local foo = 1
```

We can infer `foo` to be of type `number`, whereas the `foo` in the snippet below is inferred `any`:

```lua
local foo
foo = 1
```

However, given the second snippet in strict mode, the type checker would be able to infer `number` for `foo`.

## Structural type system

Luau's type system is structural by default, which is to say that we inspect the shape of two tables to see if they are similar enough. This was the obvious choice because Lua 5.1 is inherently structural.

```lua
type A = {x: number, y: number, z: number?}
type B = {x: number, y: number, z: number}

local a1: A = {x = 1, y = 2}        -- ok
local b1: B = {x = 1, y = 2, z = 3} -- ok

local a2: A = b1 -- ok
local b2: B = a1 -- not ok
```

## Builtin types

The Luau VM supports 10 primitive types: `nil`, `string`, `number`, `boolean`, `table`, `function`, `thread`, `userdata`, `vector`, and `buffer`. Of these, `table` and `function` are not represented by name, but have their dedicated syntax as covered in this [syntax document](syntax), `userdata` is represented by [concrete types](#roblox-types), while `vector` is not representable by name at all; other types can be specified by their name.

The type checker also provides the builtin types [`unknown`](#unknown-type), [`never`](#never-type), and [`any`](#any-type).

```lua
local s = "foo"
local n = 1
local b = true
local t = coroutine.running()

local a: any = 1
print(a.x) -- Type checker believes this to be ok, but crashes at runtime.
```

There's a special case where we intentionally avoid inferring `nil`. It's a good thing because it's never useful for a local variable to always be `nil`, thereby permitting you to assign things to it for Luau to infer that instead.

```lua
local a
local b = nil
```

### `unknown` type

`unknown` is also said to be the _top_ type, that is it's a union of all types.

```lua
local a: unknown = "hello world!"
local b: unknown = 5
local c: unknown = function() return 5 end
```

Unlike `any`, `unknown` will not allow itself to be used as a different type!

```lua
local function unknown(): unknown
    return if math.random() > 0.5 then "hello world!" else 5
end

local a: string = unknown() -- not ok
local b: number = unknown() -- not ok
local c: string | number = unknown() -- not ok
```

In order to turn a variable of type `unknown` into a different type, you must apply [type refinements](#type-refinements) on that variable.

```lua
local x = unknown()
if typeof(x) == "number" then
    -- x : number
end
```

### `never` type

`never` is also said to be the _bottom_ type, meaning there doesn't exist a value that inhabits the type `never`. In fact, it is the _dual_ of `unknown`. `never` is useful in many scenarios, and one such use case is when type refinements proves it impossible:

```lua
local x = unknown()
if typeof(x) == "number" and typeof(x) == "string" then
    -- x : never
end
```

### `any` type

`any` is just like `unknown`, except that it allows itself to be used as an arbitrary type without further checks or annotations. Essentially, it's an opt-out from the type system entirely.

```lua
local x: any = 5
local y: string = x -- no type errors here!
```

## Function types

Let's start with something simple.

```lua
local function f(x) return x end

local a: number = f(1)     -- ok
local b: string = f("foo") -- ok
local c: string = f(true)  -- not ok
```

In strict mode, the inferred type of this function `f` is `<A>(A) -> A` (take a look at [generics](#generics)), whereas in nonstrict we infer `(any) -> any`. We know this is true because `f` can take anything and then return that. If we used `x` with another concrete type, then we would end up inferring that.

Similarly, we can infer the types of the parameters with ease. By passing a parameter into *anything* that also has a type, we are saying "this and that has the same type."

```lua
local function greetingsHelper(name: string)
    return "Hello, " .. name
end

local function greetings(name)
    return greetingsHelper(name)
end

print(greetings("Alexander"))          -- ok
print(greetings({name = "Alexander"})) -- not ok
```

## Table types

From the type checker perspective, each table can be in one of three states. They are: `unsealed table`, `sealed table`, and `generic table`. This is intended to represent how the table's type is allowed to change.

### Unsealed tables

An unsealed table is a table which supports adding new properties, which updates the tables type. Unsealed tables are created using table literals. This is one way to accumulate knowledge of the shape of this table.

```lua
local t = {x = 1} -- {x: number}
t.y = 2           -- {x: number, y: number}
t.z = 3           -- {x: number, y: number, z: number}
```

However, if this local were written as `local t: { x: number } = { x = 1 }`, it ends up sealing the table, so the two assignments henceforth will not be ok.

Furthermore, once we exit the scope where this unsealed table was created in, we seal it.

```lua
local function vec2(x, y)
    local t = {}
    t.x = x
    t.y = y
    return t
end

local v2 = vec2(1, 2)
v2.z = 3 -- not ok
```

Unsealed tables are *exact* in that any property of the table must be named by the type. Since Luau treats missing properties as having value `nil`, this means that we can treat an unsealed table which does not mention a property as if it mentioned the property, as long as that property is optional.

```lua
local t = {x = 1}
local u : { x : number, y : number? } = t -- ok because y is optional
local v : { x : number, z : number } = t  -- not ok because z is not optional
```

### Sealed tables

A sealed table is a table that is now locked down. This occurs when the table type is spelled out explicitly via a type annotation, or if it is returned from a function.

```lua
local t : { x: number } = {x = 1}
t.y = 2 -- not ok
```

Sealed tables are *inexact* in that the table may have properties which are not mentioned in the type.
As a result, sealed tables support *width subtyping*, which allows a table with more properties to be used as a table with fewer properties.

```lua
type Point1D = { x : number }
type Point2D = { x : number, y : number }
local p : Point2D = { x = 5, y = 37 }
local q : Point1D = p -- ok because Point2D has more properties than Point1D
```

### Generic tables

This typically occurs when the symbol does not have any annotated types or were not inferred anything concrete. In this case, when you index on a parameter, you're requesting that there is a table with a matching interface.

```lua
local function f(t)
    return t.x + t.y
           --^   --^ {x: _, y: _}
end

f({x = 1, y = 2})        -- ok
f({x = 1, y = 2, z = 3}) -- ok
f({x = 1})               -- not ok
```

## Table indexers

These are particularly useful for when your table is used similarly to an array.

```lua
local t = {"Hello", "world!"} -- {[number]: string}
print(table.concat(t, ", "))
```

Luau supports a concise declaration for array-like tables, `{T}` (for example, `{string}` is equivalent to `{[number]: string}`); the more explicit definition of an indexer is still useful when the key isn't a number, or when the table has other fields like `{ [number]: string, n: number }`.

## Generics

The type inference engine was built from the ground up to recognize generics. A generic is simply a type parameter in which another type could be slotted in. It's extremely useful because it allows the type inference engine to remember what the type actually is, unlike `any`.

```lua
type Pair<T> = {first: T, second: T}

local strings: Pair<string> = {first="Hello", second="World"}
local numbers: Pair<number> = {first=1, second=2}
```

## Generic functions

As well as generic type aliases like `Pair<T>`, Luau supports generic functions. These are functions that, as well as their regular data parameters, take type parameters. For example, a function which reverses an array is:
```lua
function reverse(a)
  local result = {}
  for i = #a, 1, -1 do
    table.insert(result, a[i])
  end
  return result
end
```
The type of this function is that it can reverse an array, and return an array of the same type. Luau can infer this type, but if you want to be explicit, you can declare the type parameter `T`, for example:
```lua
function reverse<T>(a: {T}): {T}
  local result: {T} = {}
  for i = #a, 1, -1 do
    table.insert(result, a[i])
  end
  return result
end
```
When a generic function is called, Luau infers type arguments, for example
```lua
local x: {number} = reverse({1, 2, 3})
local y: {string} = reverse({"a", "b", "c"})
```
Generic types are used for built-in functions as well as user functions,
for example the type of two-argument `table.insert` is:
```lua
<T>({T}, T) -> ()
```

## Union types

A union type represents *one of* the types in this set. If you try to pass a union onto another thing that expects a *more specific* type, it will fail.

For example, what if this `string | number` was passed into something that expects `number`, but the passed in value was actually a `string`?

```lua
local stringOrNumber: string | number = "foo"

local onlyString: string = stringOrNumber -- not ok
local onlyNumber: number = stringOrNumber -- not ok
```

Note: it's impossible to be able to call a function if there are two or more function types in this union.

## Intersection types

An intersection type represents *all of* the types in this set. It's useful for two main things: to join multiple tables together, or to specify overloadable functions.

```lua
type XCoord = {x: number}
type YCoord = {y: number}
type ZCoord = {z: number}

type Vector2 = XCoord & YCoord
type Vector3 = XCoord & YCoord & ZCoord

local vec2: Vector2 = {x = 1, y = 2}        -- ok
local vec3: Vector3 = {x = 1, y = 2, z = 3} -- ok
```

```lua
type SimpleOverloadedFunction = ((string) -> number) & ((number) -> string)

local f: SimpleOverloadedFunction

local r1: number = f("foo") -- ok
local r2: number = f(12345) -- not ok
local r3: string = f("foo") -- not ok
local r4: string = f(12345) -- ok
```

Note: it's impossible to create an intersection type of some primitive types, e.g. `string & number`, or `string & boolean`, or other variations thereof.

Note: Luau still does not support user-defined overloaded functions. Some of Roblox and Lua 5.1 functions have different function signature, so inherently requires overloaded functions.

## Singleton types (aka literal types)

Luau's type system also supports singleton types, which means it's a type that represents one single value at runtime. At this time, both string and booleans are representable in types.

> We do not currently support numbers as types. For now, this is intentional.

```lua
local foo: "Foo" = "Foo" -- ok
local bar: "Bar" = foo   -- not ok
local baz: string = foo  -- ok

local t: true = true -- ok
local f: false = false -- ok
```

This happens all the time, especially through [type refinements](#type-refinements) and is also incredibly useful when you want to enforce program invariants in the type system! See [tagged unions](#tagged-unions) for more information.

## Variadic types

Luau permits assigning a type to the `...` variadic symbol like any other parameter:

```lua
local function f(...: number)
end

f(1, 2, 3)     -- ok
f(1, "string") -- not ok
```

`f` accepts any number of `number` values.

In type annotations, this is written as `...T`:

```lua
type F = (...number) -> ...string
```

## Type packs

Multiple function return values as well as the function variadic parameter use a type pack to represent a list of types.

When a type alias is defined, generic type pack parameters can be used after the type parameters:

```lua
type Signal<T, U...> = { f: (T, U...) -> (), data: T }
```

> Keep in mind that `...T` is a variadic type pack (many elements of the same type `T`), while `U...` is a generic type pack that can contain zero or more types and they don't have to be the same.

It is also possible for a generic function to reference a generic type pack from the generics list:

```lua
local function call<T, U...>(s: Signal<T, U...>, ...: U...)
    s.f(s.data, ...)
end
```

Generic types with type packs can be instantiated by providing a type pack:

```lua
local signal: Signal<string, (number, number, boolean)> = --

call(signal, 1, 2, false)
```

There are also other ways to instantiate types with generic type pack parameters:

```lua
type A<T, U...> = (T) -> U...

type B = A<number, ...string> -- with a variadic type pack
type C<S...> = A<number, S...> -- with a generic type pack
type D = A<number, ()> -- with an empty type pack
```

Trailing type pack argument can also be provided without parentheses by specifying variadic type arguments:

```lua
type List<Head, Rest...> = (Head, Rest...) -> ()

type B = List<number> -- Rest... is ()
type C = List<number, string, boolean> -- Rest is (string, boolean)

type Returns<T...> = () -> T...

-- When there are no type parameters, the list can be left empty
type D = Returns<> -- T... is ()
```

Type pack parameters are not limited to a single one, as many as required can be specified:

```lua
type Callback<Args..., Rets...> = { f: (Args...) -> Rets... }

type A = Callback<(number, string), ...number>
```

## Adding types for faux object oriented programs

One common pattern we see with existing Lua/Luau code is the following OO code. While Luau is capable of inferring a decent chunk of this code, it cannot pin down on the types of `self` when it spans multiple methods.

```lua
local Account = {}
Account.__index = Account

function Account.new(name, balance)
    local self = {}
    self.name = name
    self.balance = balance

    return setmetatable(self, Account)
end

-- The `self` type is different from the type returned by `Account.new`
function Account:deposit(credit)
    self.balance += credit
end

-- The `self` type is different from the type returned by `Account.new`
function Account:withdraw(debit)
    self.balance -= debit
end

local account = Account.new("Alexander", 500)
```

For example, the type of `Account.new` is `<a, b>(name: a, balance: b) -> { ..., name: a, balance: b, ... }` (snipping out the metatable). For better or worse, this means you are allowed to call `Account.new(5, "hello")` as well as `Account.new({}, {})`. In this case, this is quite unfortunate, so your first attempt may be to add type annotations to the parameters `name` and `balance`.

There's the next problem: the type of `self` is not shared across methods of `Account`, this is because you are allowed to explicitly opt for a different value to pass as `self` by writing `account.deposit(another_account, 50)`. As a result, the type of `Account:deposit` is `<a, b>(self: { balance: a }, credit: b) -> ()`. Consequently, Luau cannot infer the result of the `+` operation from `a` and `b`, so a type error is reported.

We can see there's a lot of problems happening here. This is a case where you will have to guide Luau, but using the power of top-down type inference you only need to do this in _exactly one_ place!

```lua
type AccountImpl = {
    __index: AccountImpl,
    new: (name: string, balance: number) -> Account,
    deposit: (self: Account, credit: number) -> (),
    withdraw: (self: Account, debit: number) -> (),
}

type Account = typeof(setmetatable({} :: { name: string, balance: number }, {} :: AccountImpl))

-- Only these two annotations are necessary
local Account: AccountImpl = {} :: AccountImpl
Account.__index = Account

-- Using the knowledge of `Account`, we can take in information of the `new` type from `AccountImpl`, so:
-- Account.new :: (name: string, balance: number) -> Account
function Account.new(name, balance)
    local self = {}
    self.name = name
    self.balance = balance

    return setmetatable(self, Account)
end

-- Ditto:
-- Account:deposit :: (self: Account, credit: number) -> ()
function Account:deposit(credit)
    self.balance += credit
end

-- Ditto:
-- Account:withdraw :: (self: Account, debit: number) -> ()
function Account:withdraw(debit)
    self.balance -= debit
end

local account = Account.new("Alexander", 500)
```

## Tagged unions

Tagged unions are just union types! In particular, they're union types of tables where they have at least _some_ common properties but the structure of the tables are different enough. Here's one example:

```lua
type Ok<T> = { type: "ok", value: T }
type Err<E> = { type: "err", error: E }
type Result<T, E> = Ok<T> | Err<E>
```

This `Result<T, E>` type can be discriminated by using type refinements on the property `type`, like so:

```lua
if result.type == "ok" then
    -- result is known to be Ok<T>
    -- and attempting to index for error here will fail
    print(result.value)
elseif result.type == "err" then
    -- result is known to be Err<E>
    -- and attempting to index for value here will fail
    print(result.error)
end
```

Which works out because `value: T` exists only when `type` is in actual fact `"ok"`, and `error: E` exists only when `type` is in actual fact `"err"`.

## Type refinements

When we check the type of any lvalue (a global, a local, or a property), what we're doing is we're refining the type, hence "type refinement." The support for this is arbitrarily complex, so go at it!

Here are all the ways you can refine:
1. Truthy test: `if x then` will refine `x` to be truthy.
2. Type guards: `if type(x) == "number" then` will refine `x` to be `number`.
3. Equality: `if x == "hello" then` will refine `x` to be a singleton type `"hello"`.

And they can be composed with many of `and`/`or`/`not`. `not`, just like `~=`, will flip the resulting refinements, that is `not x` will refine `x` to be falsy. 

The `assert(..)` function may also be used to refine types instead of `if/then`.

Using truthy test:
```lua
local maybeString: string? = nil

if maybeString then
    local onlyString: string = maybeString -- ok
    local onlyNil: nil = maybeString       -- not ok
end

if not maybeString then
    local onlyString: string = maybeString -- not ok
    local onlyNil: nil = maybeString       -- ok
end
```

Using `type` test:
```lua
local stringOrNumber: string | number = "foo"

if type(stringOrNumber) == "string" then
    local onlyString: string = stringOrNumber -- ok
    local onlyNumber: number = stringOrNumber -- not ok
end

if type(stringOrNumber) ~= "string" then
    local onlyString: string = stringOrNumber -- not ok
    local onlyNumber: number = stringOrNumber -- ok
end
```

Using equality test:
```lua
local myString: string = f()

if myString == "hello" then
    local hello: "hello" = myString -- ok because it is absolutely "hello"!
    local copy: string = myString   -- ok
end
```

And as said earlier, we can compose as many of `and`/`or`/`not` as we wish with these refinements:
```lua
local function f(x: any, y: any)
    if (x == "hello" or x == "bye") and type(y) == "string" then
        -- x is of type "hello" | "bye"
        -- y is of type string
    end

    if not (x ~= "hi") then
        -- x is of type "hi"
    end
end
```

`assert` can also be used to refine in all the same ways:
```lua
local stringOrNumber: string | number = "foo"

assert(type(stringOrNumber) == "string")

local onlyString: string = stringOrNumber -- ok
local onlyNumber: number = stringOrNumber -- not ok
```

## Type casts

Expressions may be typecast using `::`.  Typecasting is useful for specifying the type of an expression when the automatically inferred type is too generic.

For example, consider the following table constructor where the intent is to store a table of names:
```lua
local myTable = {names = {}}
table.insert(myTable.names, 42)         -- Inserting a number ought to cause a type error, but doesn't
```

In order to specify the type of the `names` table a typecast may be used: 

```lua
local myTable = {names = {} :: {string}}
table.insert(myTable.names, 42)         -- not ok, invalid 'number' to 'string' conversion
```

A typecast itself is also type checked to ensure that one of the conversion operands is the subtype of the other or `any`:
```lua
local numericValue = 1
local value = numericValue :: any             -- ok, all expressions may be cast to 'any'
local flag = numericValue :: boolean          -- not ok, invalid 'number' to 'boolean' conversion
```

When typecasting a variadic or the result of a function with multiple returns, only the first value will be preserved. The rest will be discarded.
```luau
function returnsMultiple(...): (number, number, number)
    print(... :: string) -- "x"
    return 1, 2, 3
end

print(returnsMultiple("x", "y", "z")) -- 1, 2, 3
print(returnsMultiple("x", "y", "z") :: number) -- 1
```

## Roblox types

Roblox supports a rich set of classes and data types, [documented here](https://developer.roblox.com/en-us/api-reference). All of them are readily available for the type checker to use by their name (e.g. `Part` or `RaycastResult`).

When one type inherits from another type, the type checker models this relationship and allows to cast a subclass to the parent class implicitly, so you can pass a `Part` to a function that expects an `Instance`.

All enums are also available to use by their name as part of the `Enum` type library, e.g. `local m: Enum.Material = part.Material`.

We can automatically deduce what calls like `Instance.new` and `game:GetService` are supposed to return:

```lua
local part = Instance.new("Part")
local basePart: BasePart = part
```

Finally, Roblox types can be refined using `IsA`:

```lua
local function getText(x : Instance) : string
    if x:IsA("TextLabel") or x:IsA("TextButton") or x:IsA("TextBox") then
        return child.Text
    end
    return ""
end
```

Note that many of these types provide some properties and methods in both lowerCase and UpperCase; the lowerCase variants are deprecated, and the type system will ask you to use the UpperCase variants instead.

## Module interactions

Let's say that we have two modules, `Foo` and `Bar`. Luau will try to resolve the paths if it can find any `require` in any scripts. In this case, when you say `script.Parent.Bar`, Luau will resolve it as: relative to this script, go to my parent and get that script named Bar.

```lua
-- Module Foo
local Bar = require(script.Parent.Bar)

local baz1: Bar.Baz = 1     -- not ok
local baz2: Bar.Baz = "foo" -- ok

print(Bar.Quux)         -- ok
print(Bar.FakeProperty) -- not ok

Bar.NewProperty = true -- not ok
```

```lua
-- Module Bar
export type Baz = string

local module = {}

module.Quux = "Hello, world!"

return module
```

There are some caveats here though. For instance, the require path must be resolvable statically, otherwise Luau cannot accurately type check it.

### Cyclic module dependencies

Cyclic module dependencies can cause problems for the type checker.  In order to break a module dependency cycle a typecast of the module to `any` may be used:
```lua
local myModule = require(MyModule) :: any
```

## Type functions

Type functions are functions that run during analysis time and operate on types, instead of runtime values. They can use the [types](#types-library) library to transform existing types or create new ones.


Here's a simplified implementation of the builtin type function `keyof`. It takes a table type and returns its property names as a [union](typecheck#union-types) of [singletons](typecheck#singleton-types-aka-literal-types).

```lua
type function simple_keyof(ty)
    -- Ignoring unions or intersections of tables for simplicity.
    if not ty:is("table") then
        error("Can only call keyof on tables.")
    end

    local union = nil

    for property in ty:properties() do
        union = if union then types.unionof(union, property) else property
    end

    return if union then union else types.singleton(nil)
end

type person = {
    name: string,
    age: number,
}
--- keys = "age" | "name"
type keys = simple_keyof<person>
```

### Type function environment

In addition to the [types](#types-library) library, type functions have access to:

* `assert`, `errror`, `print`
* `next`, `ipairs`, `pairs`
* `select`, `unpack`
* `getmetatable`, `setmetatable`
* `rawget`, `rawset`, `rawlen`, `raweq`
* `tonumber`, `tostring`
* `type`, `typeof`
* `math` library
* `table` library
* `string` library
* `bit32` library
* `utf8` library
* `buffer` library

## types library

The `types` library is used to create and transform types, and can only be used within [type functions](#type-functions).

### `types` library properties

```luau
types.any
```

The [any](typecheck#any-type) `type`.

```luau
types.unknown
```

The [unknown](typecheck#unknown-type) `type`.

```luau
types.never
```

The [never](typecheck#never-type) `type`.

```luau
types.boolean
```

The boolean `type`.

```luau
types.buffer
```

The [buffer](library#buffer-library) `type`.

```luau
types.number
```

The number `type`.

```luau
types.string
```

The string `type`.

```luau
types.thread
```

The thread `type`.

## `types` library functions

```luau
types.singleton(arg: string | boolean | nil): type
```

Returns the [singleton](typecheck#singleton-types-aka-literal-types) type of the argument.

```luau
types.negationof(arg: type): type
```

Returns an immutable negation of the argument type.

```luau
types.unionof(first: type, second: type, ...: type): type
```

Returns an immutable [union](typecheck#union-types) of two or more arguments.

```luau
types.intersectionof(first: type, second: type, ...: type): type
```

Returns an immutable [intersection](typecheck#intersection-types) of two or more arguments.

```luau
types.newtable(props: { [type]: type | { read: type?, write: type? } }?, indexer: { index: type, readresult: type, writeresult: type? }?, metatable: type?): type
```

Returns a fresh, mutable table `type`. Property keys must be string singleton `type`s. The table's metatable is set if one is provided.

```luau
types.newfunction(parameters: { head: {type}?, tail: type? }, returns: { head: {type}?, tail: type? }?, generics: {type}?): type
```

Returns a fresh, mutable function `type`, using the ordered parameters of `head` and the variadic tail of `tail`.

```luau
types.copy(arg: type): type
```

Returns a deep copy of the argument type.

```luau
types.generic(name: string?, ispack: boolean?): type
```

Creates a [generic](typecheck#generic-functions) named `name`. If `ispack` is `true`, the result is a [generic pack](typecheck#type-packs).

### `type` instance

`type` instances can have extra properties and methods described in subsections depending on its tag.

```luau
type.tag: "nil" | "unknown" | "never" | "any" | "boolean" | "number" | "string" | "singleton" | "negation" | "union" | "intersection" | "table" | "function" | "class" | "thread" | "buffer"
```

An immutable property holding the type's tag.

```luau
__eq(arg: type): boolean
```

Overrides the `==` operator to return `true` if `self` is syntactically equal to `arg`. This excludes semantically equivalent types, `true | false` is unequal to `boolean`.

```luau
type:is(arg: "nil" | "unknown" | "never" | "any" | "boolean" | "number" | "string" | "singleton" | "negation" | "union" | "intersection" | "table" | "function" | "class" | "thread" | "buffer")
```

Returns `true` if `self` has the argument as its tag.

### Singleton `type` instance

```luau
singletontype:value(): boolean | nil | "string"
```

Returns the singleton's actual value, like `true` for `types.singleton(true)`.

### Generic `type` instance

```luau
generictype:name(): string?
```

Returns the name of the [generic](typecheck#generic-functions) or `nil` if it has no name.

```luau
generictype:ispack(): boolean
```

Returns `true` if the [generic](typecheck#generic-functions) is a [pack](typecheck#type-packs), or `false` otherwise.

### Table `type` instance

```luau
tabletype:setproperty(key: type, value: type?)
```

Sets key-value pair in the table's properties, with the same type for reading from and writing to the table.

- If `key` doesn't exist in the table, does nothing.
- If `value` is `nil`, the property is removed.

```luau
tabletype:setreadproperty(key: type, value: type?)
```

Sets the key-value pair used for reading from the table's properties.

- If `key` doesn't exist in the table, does nothing.
- If `value` is `nil`, the property is removed.

```luau
tabletype:setwriteproperty(key: type, value: type?)
```

Sets the key-value pair used for writing to the table's properties.

- If `key` doesn't exist in the table, does nothing.
- If `value` is `nil`, the property is removed.

```luau
tabletype:readproperty(key: type): type?
```

Returns the type used for reading values from this property, or `nil` if the property doesn't exist.

```luau
tabletype:writeproperty(key: type): type?
```

Returns the type used for writing values to this property, or `nil` if the property doesn't exist.

```luau
tabletype:properties(): { [type]: { read: type?, write: type? } }
```

Returns a table mapping property keys to their read and write types.

```luau
tabletype:setindexer(index: type, result: type)
```

Sets the table's indexer, using the same type for reads and writes.

```luau
tabletype:setreadindexer(index: type, result: type)
```

Sets the table's indexer with the resulting read type.

```luau
tabletype:setwriteindexer(index: type, result: type)
```

Sets the table's indexer with the resulting write type.

```luau
tabletype:indexer(): { index: type, readresult: type, writeresult: type }
```

Returns the table's indexer as a table, or `nil` if it doesn't exist.

```luau
tabletype:readindexer(): { index: type, result: type }?
```

Returns the table's indexer using the result's read type, or `nil` if it doesn't exist.

```luau
tabletype:writeindexer()
```

Returns the table's indexer using the result's write type, or `nil` if it doesn't exist.

```luau
tabletype:setmetatable(arg: type)
```

Sets the table's metatable.

```luau
tabletype:metatable(): type?
```

Gets the table's metatable, or `nil` if it doesn't exist.

### Function `type` instance

```luau
functiontype:setparameters(head: {type}?, tail: type?)
```

Sets the function's parameters, with the ordered parameters in `head` and the variadic tail in `tail`.

```luau
functiontype:parameters(): { head: {type}?, tail: type? }
```

Returns the function's parameters, with the ordered parameters in `head` and the variadic tail in `tail`.

```luau
functiontype:setreturns(head: {type}?, tail: type?)
```

Sets the function's return types, with the ordered parameters in `head` and the variadic tail in `tail`.

```luau
functiontype:returns(): { head: {type}?, tail: type? }
```

Returns the function's return types, with the ordered parameters in `head` and the variadic tail in `tail`.

```luau
functiontype:generics(): {type}
```

Returns an array of the function's [generic](typecheck#generic-functions) `type`s.

```luau
functiontype:setgenerics(generics: {type}?)
```

Sets the function's [generic](typecheck#generic-functions) `type`s.

### Negation `type` instance

```luau
type:inner(): type
```

Returns the `type` being negated.

### Union `type` instance

```luau
uniontype:components(): {type}
```

Returns an array of the [unioned](typecheck#union-types) types.

### Intersection `type` instance

```luau
intersectiontype:components()
```

Returns an array of the [intersected](typecheck#intersection-types) types.

### Class `type` instance

```luau
classtype:properties(): { [type]: { read: type?, write: type? } }
```

Returns the properties of the class with their respective `read` and `write` types.

```luau
classtype:readparent(): type?
```

Returns the `read` type of the class' parent class, or returns `nil` if the parent class doesn't exist.

```luau
classtype:writeparent(): type?
```

Returns the `write` type of the class' parent class, or returns `nil` if the parent class doesn't exist.

```luau
classtype:metatable(): type?
```

Returns the class' metatable or `nil` if it doesn't exist.

```luau
classtype:indexer(): { index: type, readresult: type, writeresult: type }?
```

Returns the class' indexer, or `nil` if it doesn't exist.

```luau
classtype:readindexer(): { index: type, result: type }?
```

Returns the class' indexer using the result's read type, or `nil` if it doesn't exist.

```luau
classtype:writeindexer(): { index: type, result: type }?
```

Returns the class' indexer using the result's write type, or `nil` if it doesn't exist.
