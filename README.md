# purescript-undefined-is-not-a-problem

Handling optional record fields by using first class `undefined` (`undefined | a` "untagged union") and typesafe zero cost record coercing.

## About

The main idea behind this lib was taken from [_purescript-oneof_ library by @jvliwanag](https://github.com/jvliwanag/purescript-oneof) so all __the credits__ should __go to @jvliwanag__. _oneof_ provides a really interesting implementation of untagged unions for PureScript (especially useful in the context of FFI bindings).

I've narrowed this idea down to handle only unions with `undefined` type. I really focus on optional record fields here.

## Status

I'm about to publish. I want to use this lib in a larger project before so I would know if the API is usable enough.

## Usage

I'm going to present of two `coercing` strategies provided by this lib and discuss the difference between them.

### `NoProblem.Open`

Let me start with imports. This is a literate PureScript example (run as a part of the test suite) so we need them.

```purescript
module Test.README where

import Prelude

import Data.Undefined.NoProblem (opt, Opt, undefined, (?), (!))
import Data.Undefined.NoProblem.Closed (class Coerce, coerce) as Closed
import Data.Undefined.NoProblem.Open (class Coerce, coerce) as Open
import Effect (Effect)
import Effect.Console (logShow)
import Effect.Random (random)
import Type.Prelude (Proxy(..))
```

An API author specifies a `Record` type with all the fields which are optional (wrapped in `Opt`) so the user can skip these record properties when calling a function.

```purescript
type Options =
  { a ∷ String
  , b ∷ Opt Number
  , c ∷ Opt
    { d ∷
      { e ∷ Opt
        { f ∷ Opt String
        , g ∷ Opt Number
        , h ∷ String
        }
      }
    }
  }
```

To work with optional values we have some handy operators at our disposal:

  * a value accessor `! ∷ Opt a → a → a` which expects a default value

  * a "pseudo bind": `? ∷ Opt a → (a → Opt b) → Opt b` opertor which allows us to dive for example into optional record values.

Let's use them together with `coerce` function to build a function which work with the `Option` value internally. `coerce` is able to "fill" missing fields in a given record (recursively) with `Opt a` if that is part of the initial type and transform proper values to `Opt` ones if it is needed. This is a purely typelevel transformation.


```purescript
-- | This signature is optional
consumer ∷ ∀ r. Open.Coerce r Options ⇒ r → Number
consumer r =
  let
    -- | We should provide an info to which type we try to coerce
    opts = Open.coerce r ∷ Options

    -- | We can access and traverse optional values using "pseudoBind" function.
    -- | Side note: we can also close such a chain with `# toMaybe` easily.
    g = opts.c ? _.d.e ? _.g ! 0.0
  in
    opts.b ! 0.0 + g
```

The `Coerce` constraint checks if we can use `coerce` safely.

Now we are ready to use our function. As you can see our `argument` value lacks multiple fields and uses values directly in the places where `Opt` is really expected (like `c` should be `Opt {... }` and `g` should have type `Opt Number`):

```purescript
recordCoerce ∷ Effect Unit
recordCoerce = do
  let
    argument =
       { a: "test"
       , c:
         { d:
           { e: { g: 8.0, h: "test" }}
         }
       }

    result = consumer argument
  logShow result

```

### Optionality is just a value

It is worth nothing that optional field value is just a value. Its type is extended with additional resident - `undefined`. There are two constructor provided for `Opt`: `opt ∷ ∀ a. a → Opt a` and `undefined ∷ ∀ a. Opt a`.

You can accept or build and assemble these values on the way and pass them down to the consumers below.

```purescript
optValues :: Effect Unit
optValues = do
  -- | Under some circumstances we want
  -- | to setup part of the record
  setup ← (_ < 0.5) <$> random

  let
    { b, g } = if setup
      -- | Could be also done with `coerce`.
      then { b: opt 20.0, g: opt 25.0 }
      -- | Could be also `coerce { }`.
      else { b: undefined, g: undefined }

  logShow $
    eq
      (consumer { a: "test", b, c: { d: { e: { g, h: "test" }}}})
      (if setup then 45.0 else 0.0)
```
#### Limitiations of this approach

There is an inherent problem with coercing polymorphic types in this case. Internally I'm just not able to match a polymorphic type like `Maybe a` with expected type like `Maybe Int` and I'm not able to tell if this types are easily coercible.

In other words when you use `Open.coerce` and `Open.Coerce` then when the user provides values like `Nothing` or `[]` as a part of the argument value these pieces should be annotated.

```purescript
type OptionsWithPolymorphicValue = { x :: Opt (Array Int) }

openHandlingNonPolymorphicArray ∷ Effect Unit
openHandlingNonPolymorphicArray = do
  let
    -- | This `Array Int` signature is required
    argument = { x: [] ∷ Array Int }

  logShow $ (Open.coerce argument ∷ OptionsWithPolymorphicValue)
```

#### Pros of this approach

You can always provide an `Open.Coerce` instance for your types and allow coercing of its "internals" (please check examples in the module for `Array`, `Maybe` etc. for details).


### `NoProblem.Closed`

In general most stuff from the previous sections is relevant here as well. The small difference is in the signature of the `Colsed.coerce` function as it expects a `Proxy` value.

When you reach for this type of coercing you can expect better behavior in the case of polymorphic values. This works now without annotation for the array in `x` prop:

```purescript
closedHandlingNonPolymorphicArray ∷ Effect Unit
closedHandlingNonPolymorphicArray = do
  let
    argument = { x: [] }

    -- | Now we have still polymorhic type here: `r.x ∷ Array a`
    r = (Closed.coerce (Proxy ∷ Proxy OptionsWithPolymorphicValue) argument)

  -- | But we can easily enforce what we want when accessing a value
  logShow (r.x ! [8])
```

This strategy is a bit different because we get possibly third polymorphic type as a result from coercion which can be polymorphic but this should not be a problem in practice.

#### Limitiations

In this approach when `Closed.Coerce` instances are not able to match type like `a` with `Int` they are "pushing" the polymorphic type to the "result" type so the compiler decides if given type can be "unified" (so coercible too ;-)).
Because we are closing here an instance chain with this polymorphic case there is no way for you to provide additional instances.


### Debugging

I try to provide some debug info which should help when there is a type mismatch. For example this kind of polymorphic array value in the `z` field causes problem:

  ```purescript
  type NestedError =
    { l :: Array { x :: Opt Int, y :: Int, z :: Opt (Array  Int) }}

  x = coerce { l: [{ y: 9, z: [] }]} :: NestedError
  ```

and we can get quite informative compile time error message with property path like:

  ```shell
  Type mismatch on the path: { l."Array".z."Array" }. Expecting

  Int

  but got

  t172

  If one of the types above is a type variable like `t2` or `t37`
  it probably means that you should provide type annotation to some
  parts of your value. Something like `[] ∷ Array Int` or `Nothing ∷ Maybe String`.
  ```

But of course I'm not able to cover all types and this kind of error handling is possible for well known types.



<!--
## The Problem

### Why do you use `Record.Union` namespace?

We think about optional fields in a `Record` as a representation of sum of different types. Lets consider this type (where `Opt` marks an optional field):

  ```
  type R =
    { x ∷ Opt Int
    , y ∷ Opt Int
    }
  ```

Let's look at possible values of this hypothetical type (pseudocode):

  ```
  r ∷ Array R
  r = [ { x: 1 }, { y: 2 }, {}, { x: 1, y: 2 } ]
  ```

Of course the above won't typecheck and compile but it is not important. The thing is that we can just think of the above types in terms of a sum like:

  ```
  data R = OnlyX { x ∷ Int } | OnlyY { y ∷ Int } | None {} | XandY { x ∷ Int, y ∷ Int }
  ```
-->


<!--
But let's talk about the basics. The basic idea in `oneof` is to provide type safe casting for values of types which are members of "untagged union" type (like in _TypeScript_).

T.B.C.

When I say value of type like `Int |+| String |+| Number` we state that any value which is an `Int` a `String` or a `Number`. we can safely cast value of for example type `Number` to this.

When we extend union idea to the `Record` type (we are handing only these kind of unions here) we can nicely handle optional fields.
-->


