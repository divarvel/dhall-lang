# Binary semantics

```haskell
module Binary where

import Codec.CBOR.Term (Term(..))
import Data.List.NonEmpty (NonEmpty(..))
import Prelude hiding (Bool(..))

import Syntax
    ( Builtin(..)
    , Constant(..)
    , Expression(..)
    , File(..)
    , FilePrefix(..)
    , ImportMode(..)
    , ImportType(..)
    , Operator(..)
    , Scheme(..)
    , TextLiteral(..)
    , URL(..)
    , PathComponent(..)
    )

import qualified Data.ByteArray     as ByteArray
import qualified Data.List          as List
import qualified Data.List.NonEmpty as NonEmpty
import qualified Data.Time          as Time
import qualified Data.Ord           as Ord
import qualified GHC.Float          as Float
import qualified Numeric.Half       as Half
import qualified Prelude
```

This document formalizes the semantics for encoding and decoding Dhall
expressions to and from a binary representation

* [Motivation](#motivation)
* [CBOR](#cbor)
* [CBOR expressions](#cbor-expressions)
* [Encoding judgment](#encoding-judgment)
    * [Variables](#variables)
    * [Built-in constants](#built-in-constants)
    * [Function application](#function-application)
    * [Functions](#functions)
    * [Operators](#operators)
    * [`List`](#list)
    * [`Optional`](#optional)
    * [`merge`-expressions](#merge-expressions)
    * [Records](#records)
    * [Unions](#unions)
    * [`Bool`](#bool)
    * [`Natural`](#natural)
    * [`Integer`](#integer)
    * [`Double`](#double)
    * [`Text`](#text)
    * [`assert`](#assert)
    * [Imports](#imports)
    * [`let`-expressions](#let-expressions)
    * [Type annotations](#type-annotations)
    * [`with`-expressions](#with-expressions)
    * [`Date` / `Time` / `TimeZone`](#date--time--timezone)
* [Decoding judgment](#decoding-judgment)
    * [CBOR Tags](#cbor-tags)
    * [Built-in constants](#built-in-constants-1)
    * [Variables](#variables-1)
    * [Function application](#function-application-1)
    * [Functions](#functions-1)
    * [Operators](#operators-1)
    * [`List`](#list-1)
    * [`Optional`](#optional-1)
    * [`merge`-expressions](#merge-expressions-1)
    * [Records](#records-1)
    * [Unions](#unions-1)
    * [`Bool`](#bool-1)
    * [`Natural`](#natural-1)
    * [`Integer`](#integer-1)
    * [`Double`](#double-1)
    * [`Text`](#text-1)
    * [`assert`](#assert-1)
    * [Imports](#imports-1)
    * [`let`-expressions](#let-expressions-1)
    * [Type annotations](#type-annotations-1)
    * [`with`-expressions](#with-expressions-1)
    * [`Date` / `Time` / `TimeZone`](#date--time--timezone-1)

## Motivation

Dhall's import system requires a standard way to convert expressions to and from
a binary representation, for two reasons:

* Users can import expressions protected by a "semantic integrity check",
  which is a SHA-256 hash of the binary representation of an expression's
  normal form

* Interpreters can locally cache imported expressions if the user protects them
  with a semantic integrity check.  The local cache stores expressions using
  the SHA-256 hash as the lookup key and the binary representation of the
  expression as the cached value.

## CBOR

Dhall's semantics for binary serialization are defined in terms of the
Concise Binary Object Representation (CBOR)
([RFC 7049](https://tools.ietf.org/html/rfc7049)).  This section first specifies
a simplified grammar for the subset of CBOR that Dhall uses
(i.e. "CBOR expressions") and then specifies how to convert Dhall expressions to
and from these CBOR expressions.

The `encode` judgments in this section specify how to convert a Dhall
expression to a CBOR expression.  Once you have a CBOR expression you can
serialize that CBOR expression to a binary representation according to RFC 7049.

The `decode` judgments in this section specify how to convert a CBOR
expression to a Dhall expression.  Once you deserialize a CBOR expression from
a binary representation then you can further decode that CBOR expression to a
Dhall expression.

For efficiency reasons, an implementation MAY elect to not go through an
intermediate CBOR expression and instead serialize to or deserialize from the
binary representation directly.  These semantics only specify how to perform the
conversion through an intermediate CBOR expression for simplicity.

## CBOR expressions

The following notation will be used for CBOR expressions in serialization
judgments:

```
e =   n              ; Unsigned integer    (Section 2.1, Major type = 0)
  /  -n              ; Negative integer    (Section 2.1, Major type = 1)
  /  b"…"            ; Byte string         (Section 2.1, Major type = 2)
  /  "…"             ; Text                (Section 2.1, Major type = 3)
  /  [ e, es… ]      ; Heterogeneous array (Section 2.1, Major type = 4)
  /  { e = e, es… }  ; Heterogeneous map   (Section 2.1, Major type = 5)
  /  False           ; False               (Section 2.3, Value = 20)
  /  True            ; True                (Section 2.3, Value = 21)
  /  null            ; Null                (Section 2.3, Value = 22)
  /  n.n_h           ; Half float          (Section 2.3, Value = 25)
  /  n.n_s           ; Single float        (Section 2.3, Value = 26)
  /  n.n             ; Double float        (Section 2.3, Value = 27)
  /  nn              ; Unsigned bignum     (Section 2.4, Tag = 2)
  / -nn              ; Negative bignum     (Section 2.4, Tag = 3)
  /  m*10^e          ; Decimal fraction    (Section 2.4, Tag = 4)
  / CBORTag n e      ; CBOR tag            (Section 2.4, Tag = n)
```

## Expression labels

The CBOR encoding scheme uses a convention where expressions are
encoded as CBOR arrays, with the first element an integer that
identifies the type of expression.  In this document we call this kind
of integer a "label".  This labelling scheme is specific to Dhall, and
should not be confused with CBOR tags, as documented in
[RFC7049 section 2.4](https://tools.ietf.org/html/rfc7049#section-2.4).

## Encoding judgment

You can encode a Dhall expression using the following judgment:

    encode(dhall) = cbor

... where:

* `dhall` (the input) is a Dhall expression
* `cbor` (the output) is a CBOR expression

```haskell
encode :: Expression -> Term
```

The encoding logic includes several optimizations for more compactly encoding
expressions that are fully resolved and αβ-normalized because expressions are
fully interpreted before they are hashed or cached.  For example, the encoding
uses labels less than 24 for language features that survive both import resolution
and normalization (such as a function type) since those labels fit in one byte.
Language features that don't survive full interpretation (such as a `let`
expression or an import) use labels of 24 or above.  Similarly, the encoding using
a more compact representation for variables named `_` because no other variable
names remain after α-normalization.

Do not construe these optimizations to mean that you need to resolve imports
or αβ-normalize expressions.  You may encode or decode expressions that have
not been resolved in any way.

### Variables

Encode a variable named `"_"` as its index, using the smallest numeric
representation available:


    ───────────────  ; n < 2^64
    encode(_@n) = n


    ────────────────  ; 2^64 <= n
    encode(_@n) = nn


```haskell
encode (Variable "_" n)
    | n < 0xFFFFFFFFFFFFFFFF = TInt     (fromIntegral n)
    | otherwise              = TInteger (fromIntegral n)
```

This optimization takes advantage of the fact that α-normalized expressions
only use variables named `_`.  Encoding an α-normalized expression is equivalent
to using De-Bruijn indices to label variables.

The reason for this optimization is because expressions are commonly
α-normalized before encoding them, such as when computing their semantic
integrity check and also when caching them.

Encode a variable that is not named `"_"` as a two-element CBOR array where
the first element is the identifier and the second element is the encoded
index (using the smallest numeric representation available):


    ────────────────────────  ; n < 2^64
    encode(x@n) = [ "x", n ]


    ─────────────────────────  ; 2^64 <= nn
    encode(x@n) = [ "x", nn ]


```haskell
encode (Variable x n)
    | n < 0xFFFFFFFFFFFFFFFF = TList [ TString x, TInt (fromIntegral n) ]
    | otherwise              = TList [ TString x, TInteger (fromIntegral n) ]
```

### Built-in constants

Encode all built-in constants (except boolean values) as naked strings
matching their identifier.


    ───────────────────────────────────────
    encode(Natural/build) = "Natural/build"


    ─────────────────────────────────────
    encode(Natural/fold) = "Natural/fold"


    ─────────────────────────────────────────
    encode(Natural/isZero) = "Natural/isZero"


    ─────────────────────────────────────
    encode(Natural/even) = "Natural/even"


    ───────────────────────────────────
    encode(Natural/odd) = "Natural/odd"


    ───────────────────────────────────────────────
    encode(Natural/toInteger) = "Natural/toInteger"


    ─────────────────────────────────────
    encode(Natural/show) = "Natural/show"


    ─────────────────────────────────────────────
    encode(Natural/subtract) = "Natural/subtract"


    ─────────────────────────────────────────────
    encode(Integer/toDouble) = "Integer/toDouble"


    ─────────────────────────────────────
    encode(Integer/show) = "Integer/show"


    ─────────────────────────────────────────
    encode(Integer/negate) = "Integer/negate"


    ───────────────────────────────────────
    encode(Integer/clamp) = "Integer/clamp"


    ───────────────────────────────────
    encode(Double/show) = "Double/show"


    ─────────────────────────────────
    encode(List/build) = "List/build"


    ───────────────────────────────
    encode(List/fold) = "List/fold"


    ───────────────────────────────────
    encode(List/length) = "List/length"


    ───────────────────────────────
    encode(List/head) = "List/head"


    ───────────────────────────────
    encode(List/last) = "List/last"


    ─────────────────────────────────────
    encode(List/indexed) = "List/indexed"


    ─────────────────────────────────────
    encode(List/reverse) = "List/reverse"


    ───────────────────────────────
    encode(Text/show) = "Text/show"


    ─────────────────────────────────────
    encode(Text/replace) = "Text/replace"


    ───────────────────────────────
    encode(Date/show) = "Date/show"


    ───────────────────────────────
    encode(Time/show) = "Time/show"


    ───────────────────────────────────────
    encode(TimeZone/show) = "TimeZone/show"


    ─────────────────────
    encode(Bool) = "Bool"


    ─────────────────────────────
    encode(Optional) = "Optional"


    ─────────────────────
    encode(None) = "None"


    ───────────────────────────
    encode(Natural) = "Natural"


    ───────────────────────────
    encode(Integer) = "Integer"


    ─────────────────────────
    encode(Double) = "Double"


    ─────────────────────
    encode(Text) = "Text"


    ───────────────────────
    encode(Bytes) = "Bytes"


    ─────────────────────
    encode(List) = "List"


    ─────────────────────
    encode(Date) = "Date"


    ─────────────────────
    encode(Time) = "Time"


    ─────────────────────────────
    encode(TimeZone) = "TimeZone"


    ─────────────────────
    encode(Type) = "Type"


    ─────────────────────
    encode(Kind) = "Kind"


    ─────────────────────
    encode(Sort) = "Sort"


```haskell
encode (Builtin NaturalBuild    ) = TString "Natural/build"
encode (Builtin NaturalFold     ) = TString "Natural/fold"
encode (Builtin NaturalIsZero   ) = TString "Natural/isZero"
encode (Builtin NaturalEven     ) = TString "Natural/even"
encode (Builtin NaturalOdd      ) = TString "Natural/odd"
encode (Builtin NaturalToInteger) = TString "Natural/toInteger"
encode (Builtin NaturalShow     ) = TString "Natural/show"
encode (Builtin NaturalSubtract ) = TString "Natural/subtract"
encode (Builtin IntegerToDouble ) = TString "Integer/toDouble"
encode (Builtin IntegerShow     ) = TString "Integer/show"
encode (Builtin IntegerNegate   ) = TString "Integer/negate"
encode (Builtin IntegerClamp    ) = TString "Integer/clamp"
encode (Builtin DoubleShow      ) = TString "Double/show"
encode (Builtin ListBuild       ) = TString "List/build"
encode (Builtin ListFold        ) = TString "List/fold"
encode (Builtin ListLength      ) = TString "List/length"
encode (Builtin ListHead        ) = TString "List/head"
encode (Builtin ListLast        ) = TString "List/last"
encode (Builtin ListIndexed     ) = TString "List/indexed"
encode (Builtin ListReverse     ) = TString "List/reverse"
encode (Builtin TextShow        ) = TString "Text/show"
encode (Builtin TextReplace     ) = TString "Text/replace"
encode (Builtin DateShow        ) = TString "Date/show"
encode (Builtin TimeShow        ) = TString "Time/show"
encode (Builtin TimeZoneShow    ) = TString "TimeZone/show"
encode (Builtin Bool            ) = TString "Bool"
encode (Builtin Optional        ) = TString "Optional"
encode (Builtin None            ) = TString "None"
encode (Builtin Natural         ) = TString "Natural"
encode (Builtin Integer         ) = TString "Integer"
encode (Builtin Double          ) = TString "Double"
encode (Builtin Text            ) = TString "Text"
encode (Builtin Bytes           ) = TString "Bytes"
encode (Builtin List            ) = TString "List"
encode (Builtin Date            ) = TString "Date"
encode (Builtin Time            ) = TString "Time"
encode (Builtin TimeZone        ) = TString "TimeZone"
encode (Constant Type           ) = TString "Type"
encode (Constant Kind           ) = TString "Kind"
encode (Constant Sort           ) = TString "Sort"
```

### Function application

Function application is encoded as a heterogeneous array where a function
applied to multiple arguments is stored within a single array:


    encode(f₀) = f₁   encode(a₀) = a₁   encode(b₀) = b₁   …
    ───────────────────────────────────────────────────────
    encode(f₀ a₀ b₀ …) = [ 0, f₁, a₁, b₁, … ]


```haskell
encode expression@(Application _ _) = loop [] expression
  where
    loop arguments (Application function a₀) = loop (a₁ : arguments) function
      where
        a₁ = encode a₀
    loop arguments f₀ = TList ([ TInt 0, f₁ ] ++ arguments)
      where
        f₁ = encode f₀
```

### Functions

Functions that bind variables named `_` have a more compact representation:


    encode(A₀) = A₁   encode(b₀) = b₁
    ──────────────────────────────────────
    encode(λ(_ : A₀) → b₀) = [ 1, A₁, b₁ ]


```haskell
encode (Lambda "_" _A₀ b₀) = TList [ TInt 1, _A₁, b₁ ]
  where
    _A₁ = encode _A₀

    b₁ = encode b₀
```

... than functions that bind variables of other names:


    encode(A₀) = A₁   encode(b₀) = b₁
    ───────────────────────────────────────────  ; x ≠ "_"
    encode(λ(x : A₀) → b₀) = [ 1, "x", A₁, b₁ ]


```haskell
encode (Lambda x _A₀ b₀) = TList [ TInt 1, TString x, _A₁, b₁ ]
  where
    _A₁ = encode _A₀

    b₁ = encode b₀
```

Function types that bind variables named `_` also have a more compact
representation:


    encode(A₀) = A₁   encode(B₀) = B₁
    ──────────────────────────────────────
    encode(∀(_ : A₀) → B₀) = [ 2, A₁, B₁ ]


```haskell
encode (Forall "_" _A₀ _B₀) = TList [ TInt 2, _A₁, _B₁ ]
  where
    _A₁ = encode _A₀

    _B₁ = encode _B₀
```

... than function types that bind variables of other names:


    encode(A₀) = A₁   encode(B₀) = B₁
    ───────────────────────────────────────────  ; x ≠ "_"
    encode(∀(x : A₀) → B₀) = [ 2, "x", A₁, B₁ ]


```haskell
encode (Forall x _A₀ _B₀) = TList [ TInt 2, TString x, _A₁, _B₁ ]
  where
    _A₁ = encode _A₀

    _B₁ = encode _B₀
```

### Operators

Operators are encoded as integer labels alongside their two arguments:


    encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────
    encode(l₀ || r₀) = [ 3, 0, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────
    encode(l₀ && r₀) = [ 3, 1, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────
    encode(l₀ == r₀) = [ 3, 2, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────
    encode(l₀ != r₀) = [ 3, 3, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ──────────────────────────────────
    encode(l₀ + r₀) = [ 3, 4, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ──────────────────────────────────
    encode(l₀ * r₀) = [ 3, 5, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────
    encode(l₀ ++ r₀) = [ 3, 6, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ──────────────────────────────────
    encode(l₀ # r₀) = [ 3, 7, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ──────────────────────────────────
    encode(l₀ ∧ r₀) = [ 3, 8, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ──────────────────────────────────
    encode(l₀ ⫽ r₀) = [ 3, 9, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────
    encode(l₀ ⩓ r₀) = [ 3, 10, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────
    encode(l₀ ? r₀) = [ 3, 11, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ─────────────────────────────────────
    encode(l₀ === r₀) = [ 3, 12, l₁, r₁ ]


    encode(l₀) = l₁   encode(r₀) = r₁
    ────────────────────────────────────
    encode(l₀ :: r₀) = [ 3, 13, l₁, r₁ ]


```haskell
encode (Operator l₀ op r₀) = TList [ TInt 3, TInt opcode, l₁, r₁ ]
  where
    l₁ = encode l₀

    r₁ = encode r₀

    opcode = case op of
        Or                 ->  0
        And                ->  1
        Equal              ->  2
        NotEqual           ->  3
        Plus               ->  4
        Times              ->  5
        TextAppend         ->  6
        ListAppend         ->  7
        CombineRecordTerms ->  8
        Prefer             ->  9
        CombineRecordTypes -> 10
        Alternative        -> 11
        Equivalent         -> 12

encode (Completion l₀ r₀) = TList [TInt 3, TInt 13, l₁, r₁ ]
  where
    l₁ = encode l₀

    r₁ = encode r₀
```

### `List`

Empty `List`s only store their type:


    encode(T₀) = T₁
    ────────────────────────────────
    encode([] : List T₀) = [ 4, T₁ ]


```haskell
encode (EmptyList (Application (Builtin List) _T₀)) = TList [ TInt 4, _T₁ ]
  where
    _T₁ = encode _T₀
```

If the type annotation is not of the form `List T`:


    encode(T₀) = T₁
    ────────────────────────────
    encode([] : T₀) = [ 28, T₁ ]


```haskell
encode (EmptyList _T₀) = TList [ TInt 28, _T₁ ]
  where
    _T₁ = encode _T₀
```

Non-empty `List`s don't store their type, but do store their elements inline:


    encode(a₀) = a₁   encode(b₀) = b₁
    ──────────────────────────────────────────────
    encode([ a₀, b₀, … ]) = [ 4, null, a₁, b₁, … ]


```haskell
encode (NonEmptyList as₀) = TList ([ TInt 4, TNull ] ++ as₁)
  where
    as₁ = map encode (NonEmpty.toList as₀)
```

### `Some`

`Some` expressions store the type (if present) and their value:


    encode(t₀) = t₁
    ─────────────────────────────────
    encode(Some t₀) = [ 5, null, t₁ ]


```haskell
encode (Some t₀) = TList [ TInt 5, TNull, t₁ ]
  where
    t₁ = encode t₀
```

### `merge` expressions

`merge` expressions differ in their encoding depending on whether or not they
have a type annotation:


    encode(t₀) = t₁   encode(u₀) = u₁
    ───────────────────────────────────
    encode(merge t₀ u₀) = [ 6, t₁, u₁ ]


    encode(t₀) = t₁   encode(u₀) = u₁   encode(T₀) = T₁
    ───────────────────────────────────────────────────
    encode(merge t₀ u₀ : T₀) = [ 6, t₁, u₁, T₁ ]


```haskell
encode (Merge t₀ u₀ Nothing) = TList [ TInt 6, t₁, u₁ ]
  where
    t₁ = encode t₀

    u₁ = encode u₀

encode (Merge t₀ u₀ (Just _T₀)) = TList [ TInt 6, t₁, u₁, _T₁ ]
  where
    t₁ = encode t₀

    u₁ = encode u₀

    _T₁ = encode _T₀
```

### `toMap` expressions

`toMap` expressions differ in their encoding depending on whether or not they
have a type annotation:


    encode(t₀) = t₁
    ─────────────────────────────
    encode(toMap t₀) = [ 27, t₁ ]


    encode(t₀) = t₁   encode(T₀) = T₁
    ──────────────────────────────────────
    encode(toMap t₀ : T₀) = [ 27, t₁, T₁ ]


```haskell
encode (ToMap t₀ Nothing) = TList [ TInt 27, t₁ ]
  where
    t₁ = encode t₀

encode (ToMap t₀ (Just _T₀)) = TList [ TInt 27, t₁, _T₁ ]
  where
    t₁ = encode t₀

    _T₁ = encode _T₀
```

### `showConstructor` expressions

    encode(t₀) = t₁
    ─────────────────────────────
    encode(showConstructor t₀) = [ 34, t₁ ]


```haskell
encode (ShowConstructor t₀) = TList [ TInt 34, t₁ ]
  where
    t₁ = encode t₀
```

### Records

Dhall record types translate to CBOR maps:


    encode(T₀) = T₁   …
    ──────────────────────────────────────────────
    encode({ x : T₀, … }) = [ 7, { "x" = T₁, … } ]


```haskell
encode (RecordType xTs₀) = TList [ TInt 7, TMap xTs₁ ]
  where
    xTs₁ = map adapt (List.sortBy (Ord.comparing fst) xTs₀)

    adapt (x, _T₀) = (TString x, _T₁)
      where
        _T₁ = encode _T₀
```

Dhall record literals translate to CBOR maps:


    encode(t₀) = t₁   …
    ──────────────────────────────────────────────
    encode({ x = t₀, … }) = [ 8, { "x" = t₁, … } ]


```haskell
encode (RecordLiteral xts₀) = TList [ TInt 8, TMap xts₁ ]
  where
    xts₁ = map adapt (List.sortBy (Ord.comparing fst) xts₀)

    adapt (x, _T₀) = (TString x, _T₁)
      where
        _T₁ = encode _T₀
```

Note: the record fields should be sorted before translating them to CBOR maps.


Field access:


    encode(t₀) = t₁
    ─────────────────────────────
    encode(t₀.x) = [ 9, t₁, "x" ]


```haskell
encode (Field t₀ x) = TList [ TInt 9, t₁, TString x ]
  where
    t₁ = encode t₀
```

... is encoded differently than record projection:


    encode(t₀) = t₁   …
    ────────────────────────────────────────────────────
    encode(t₀.{ x, y, … }) = [ 10, t₁, "x", "y", … ]


```haskell
encode (ProjectByLabels t₀ xs) = TList ([ TInt 10, t₁ ] ++ map TString xs)
  where
    t₁ = encode t₀
```

Record projection by type is encoded as follows:


    encode(t₀) = t₁   encode(T₀) = T₁
    ────────────────────────────────────
    encode(t₀.(T₀)) = [ 10, t₁, [ T₁ ] ]


```haskell
encode (ProjectByType t₀ _T₀) = TList [ TInt 10, t₁, TList [ _T₁ ] ]
  where
    t₁ = encode t₀

    _T₁ = encode _T₀
```

### Unions

Dhall union types translate to CBOR maps:


    encode(T₀) = T₁   …
    ────────────────────────────────────────────────────────────────
    encode(< x : T₀ | y | … >) = [ 11, { "x" = T₁, "y" = null, … } ]


```haskell
encode (UnionType xTs₀) = TList [ TInt 11, TMap xTs₁ ]
  where
    xTs₁ = map adapt (List.sortBy (Ord.comparing fst) xTs₀)

    adapt (x, Just _T₀) = (TString x, _T₁)
      where
        _T₁ = encode _T₀
    adapt (x, Nothing) = (TString x, TNull)
```

Union constructors (`U.x`) are encoded according to the rule for record field
accesses above.

The (now-removed) union literal syntax used to be encoded using a leading 12:


    encode(t₀) = t₁   encode(T₀) = T₁   …
    ──────────────────────────────────────────────────────────────────────────────────  ; OBSOLETE judgment
    encode(< x = t₀ | y : T₀ | z | … >) = [ 12, "x", t₁, { "y" = T₁, "z" = null, … } ]


The (now-removed) `constructors` keyword used to be encoded using a leading 13:


    encode(u₀) = u₁
    ───────────────────────────────────────  ; OBSOLETE judgment
    encode(constructors u₀) = [ 13, u₁]


Avoid reusing the numbers 12 and 13 as long as possible until we run out of
numbers less than 24, mainly to avoid old encoded expressions containing union
literals or the `constructors` keyword from being decoded as a newer type of
expression.  Once we run out of numbers, first reassign 13 before reassigning
12, because `constructors` was removed from the language before union literals
were.

### `Bool`

Encode Boolean literals using CBOR's built-in support for Boolean values:


    ───────────────────────
    encode(True) = True


    ─────────────────────────
    encode(False) = False


```haskell
encode (Builtin True) = TBool Prelude.True

encode (Builtin False) = TBool Prelude.False
```

`if` expressions are encoded with a label:


    encode(t₀) = t₁   encode(l₀) = l₁   encode(r₀) = r₁
    ───────────────────────────────────────────────────────────────
    encode(if t₀ then l₀ else r₀) = [ 14, t₁, l₁, r₁ ]


```haskell
encode (If t₀ l₀ r₀) = TList [ TInt 14, t₁, l₁, r₁ ]
  where
    t₁ = encode t₀

    l₁ = encode l₀

    r₁ = encode r₀
```

### `Natural`

Encode `Natural` literals using the smallest available numeric representation:


    ─────────────────────────  ; n < 2^64
    encode(n) = [ 15, n ]


    ──────────────────────────  ; 2^64 <= nn
    encode(n) = [ 15, nn ]


```haskell
encode (NaturalLiteral n)
    | n < 0xFFFFFFFFFFFFFFFF = TList [ TInt 15, TInt (fromIntegral n) ]
    | otherwise              = TList [ TInt 15, TInteger (fromIntegral n) ]
```

### `Integer`

Encode `Integer` literals using the smallest available numeric representation:


    ────────────────────────────  ; ±n < -2^64
    encode(±n) = [ 16, -nn ]


    ───────────────────────────  ; -2^64 <= ±n < 0
    encode(±n) = [ 16, -n ]


    ──────────────────────────  ; 0 <= ±n < 2^64
    encode(±n) = [ 16, n ]


    ───────────────────────────  ; 2^64 <= ±n
    encode(±n) = [ 16, nn ]


```haskell
encode (IntegerLiteral n)
    | n < -0xFFFFFFFFFFFFFFFF = TList [ TInt 16, TInteger n ]
    | n <  0                  = TList [ TInt 16, TInt (fromInteger n) ]
    | n <  0xFFFFFFFFFFFFFFFF = TList [ TInt 16, TInt (fromInteger n) ]
    | otherwise               = TList [ TInt 16, TInteger n ]
```

### `Double`

CBOR has 16-bit, 32-bit, and 64-bit IEEE 754 floating point representations. As
recommended by the [working draft of the CBOR Working Group][rfc7049bis],
encode a `Double` literal using the shortest floating point encoding that
preserves its value.

[rfc7049bis]: https://tools.ietf.org/html/draft-ietf-cbor-7049bis-04#section-4.6


    ─────────────────────────────  ; toDouble(toHalf(n.n)) = n.n
    encode(n.n) = n.n_h


    ─────────────────────────────  ; toDouble(toHalf(n.n)) ≠ n.n AND toDouble(toSingle(n.n)) = n.n
    encode(n.n) = n.n_s


    ─────────────────────────────  ; toDouble(toSingle(n.n)) ≠ n.n
    encode(n.n) = n.n


```haskell
encode (DoubleLiteral n)
    | isNaN n                                     = THalf n_s
    | Float.float2Double (Half.fromHalf n_h) == n = THalf n_s
    | Float.float2Double n_s == n                 = TFloat n_s
    | otherwise                                   = TDouble n
  where
    n_s = Float.double2Float n

    n_h = Half.toHalf n_s
```

Note in particular the encoding of these special values:


    ───────────────────────────  ; Even though NaN does not equal itself, we
    encode(NaN) = n.n_h(0x7e00)  ; still encode it using half-precision


    ────────────────────────────────
    encode(Infinity) = n.n_h(0x7c00)


    ─────────────────────────────────
    encode(-Infinity) = n.n_h(0xfc00)


    ────────────────────────────
    encode(+0.0) = n.n_h(0x0000)


    ────────────────────────────
    encode(-0.0) = n.n_h(0x8000)



These values ensure identical semantic hashes on different platforms.


### `Text`

Encode `Text` literals as an alternation between their text chunks and any
interpolated expressions:


    encode(b₀) = b₁   encode(d₀) = d₁   …   encode(y₀) = y₁
    ───────────────────────────────────────────────────────────────────────────────────
    encode("a${b₀}c${d}e…x${y₀}z") = [ 18, "a", b₁, "c", d₁, "e", …, "x", y₁, "z" ]


```haskell
encode (TextLiteral (Chunks xys₀ z)) = TList (TInt 18 : xys₁ ++ [ TString z ])
  where
    xys₁ = do
        (x, y₀) <- xys₀

        let y₁ = encode y₀

        [ TString x, y₁ ]
```

In other words: the amount of encoded elements is always an odd number, with the
odd elements being strings and the even ones being interpolated expressions.
Note: this means that the first and the last encoded elements are always strings,
even if they are empty strings.

### `Bytes`

Encode `Bytes` literals by decoding it from its encoded representation to a CBOR
major type 2 byte string:


    base16decode(0x"0123456789abcdef") = bytes
    ───────────────────────────────────────────────
    encode(0x"0123456789abcdef") = [ 33, b"bytes" ]


The `base16decode` judgement stands in for the base-16 decoding algorithm
specified in
[RFC4648 - Section 8](https://tools.ietf.org/html/rfc4648#section-8), treated
as a pure function from text to a byte array.

```haskell
encode (BytesLiteral xs) = TList [ TInt 33, TBytes xs ]
```

### `assert`

An assertion encoded its own annotation (regardless of whether the annotation is
an equivalence or not):


    encode(T₀) = T₁
    ────────────────────────────────
    encode(assert : T₀) = [ 19, T₁ ]


```haskell
encode (Assert _T₀) = TList [ TInt 19, _T₁ ]
  where
    _T₁ = encode _T₀
```

### Imports

Imports are encoded as a list where the first three elements are always:

* `24` - To signal that this expression is an import
* An optional integrity check
    * The integrity check is represented as `null` if absent
    * The integrity check is b"[multihash][]" if present
* An optional import type (such as `as Text`)
    * The import type is `0` for when importing a Dhall expression (the default)
    * The import type is `1` for importing `as Text`
    * The import type is `2` for importing `as Location`
    * The import type is `3` for importing `as Bytes`

For example, if an import does not specify an integrity check or import type
then the CBOR expression begins with:

    [ 24, null, 0, … ]

After that a URL import contains the following elements:

* The scheme, which is 0 (if the scheme is http) or 1 (if the scheme is https)
* An optional custom headers specification
    * The specification is `null` if absent
* The authority
    * This includes user information and port if present
    * This does not include the preceding "//" or the following "/"
    * For example, the authority of `http://user@host:port/foo` is encoded as
      `"user@host:port"`
* Then one element per path component
    * The encoded path components do not include their separating slashes
    * For example, `/foo/bar/baz` is stored as `…, "foo", "bar", "baz", …`
    * There is always at least one path component.  If the source URL has no
      path, [normalize it to `/`][RFC7230§2.7.3] before encoding.
* Then one element for the query component
    * If there is no query component then it is encoded as `null`
    * If there is a query component then it is stored without the `?`
    * A query component with internal `&` separators is still one element
    * For example `?foo=1&bar=true` is stored as `"foo=1&bar=true"`

[RFC7230§2.7.3]: https://tools.ietf.org/html/rfc7230#section-2.7.3

The full rules are:


    ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    encode(http://authority/path₀/path₁/…/file?query) = [ 24, null, 0, 0, null, "authority", "path₀", "path₁", …, "file", "query" ]


    ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    encode(http://authority/path₀/path₁/…/file) = [ 24, null, 0, 0, null, "authority", "path₀", "path₁", …, "file", null ]



    ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    encode(https://authority/path₀/path₁/…/file?query) = [ 24, null, 0, 1, null, "authority", "path₀", "path₁", …, "file", "query" ]


    ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    encode(https://authority/path₀/path₁/…/file) = [ 24, null, 0, 1, null, "authority", "path₀", "path₁", …, "file", null ]


If you import `using headers`, then the fourth element contains the import
header:


    encode(http://authority directory file) = [ 24, x, y, 0, null, xs… ]
    encode(headers) = h
    ───────────────────────────────────────────────────────────────────────────────
    encode(http://authority directory file using headers) = [ 24, x, y, 0, h, xs… ]


    encode(https://authority directory file) = [ 24, x, y, 1, null, xs… ]
    encode(headers) = h
    ────────────────────────────────────────────────────────────────────────────────
    encode(https://authority directory file using headers) = [ 24, x, y, 1, h, xs… ]


Absolute file paths are tokenized in the same way:


    ─────────────────────────────────────────────────────────────────────────────
    encode(/path₀/path₁/…/file) = [ 24, null, 0, 2, "path₀", "path₁", …, "file" ]


Each path type is treated as another "scheme" (i.e. they are distinguished by
the third element):


    ──────────────────────────────────────────────────────────────────────────────
    encode(./path₀/path₁/…/file) = [ 24, null, 0, 3, "path₀", "path₁", …, "file" ]


    ───────────────────────────────────────────────────────────────────────────────
    encode(../path₀/path₁/…/file) = [ 24, null, 0, 4, "path₀", "path₁", …, "file" ]


    ──────────────────────────────────────────────────────────────────────────────
    encode(~/path₀/path₁/…/file) = [ 24, null, 0, 5, "path₀", "path₁", …, "file" ]


Environment variables are also treated as another scheme:


    ───────────────────────────────────────
    encode(env:x) = [ 24, null, 0, 6, "x" ]


The `missing` keyword is also treated as another import type:


    ────────────────────────────────────
    encode(missing) = [ 24, null, 0, 7 ]


If an import has an integrity check, store that in the second element as a byte
string of the [multihash][]. Implementors do not need to be concerned
with full multihash at this point since Dhall only supports sha256,
so this rule suffices:


    encode(import) = [ 24, null, x, xs… ]
    base16decode(base16Hash) = rawHash
    ─────────────────────────────────────────────────────────────────────────────
    encode(import sha256:base16Hash) = [ 24, b"\x12\x20rawHash", x, xs… ]



If you import `as Text`, then the third element encoding the import type is `1`
instead of `0`:


    encode(import) = [ 24, x, 0, xs… ]
    ──────────────────────────────────────────
    encode(import as Text) = [ 24, x, 1, xs… ]


If you import `as Bytes`, then the third element encoding the import type is `3`
instead of `0`:


    encode(import) = [ 24, x, 0, xs… ]
    ───────────────────────────────────────────
    encode(import as Bytes) = [ 24, x, 3, xs… ]


If you import `as Location`, then the third element encoding the import type is
`2` instead of `0`:


    encode(import) = [ 24, x, 0, xs… ]
    ──────────────────────────────────────────
    encode(import as Location) = [ 24, x, 2, xs… ]


```haskell
encode (Import importType₀ importMode₀ hash₀) =
    TList ([ TInt 24, hash₁, importMode₁ ] <> importType₁)
  where
    hash₁ = case hash₀ of
        Just digest -> TBytes ("\x12\x20" <> ByteArray.convert digest)
        Nothing     -> TNull

    importMode₁ = case importMode₀ of
        Code     -> TInt 0
        RawText  -> TInt 1
        RawBytes -> TInt 3
        Location -> TInt 2

    importType₁ = case importType₀ of
        Remote (URL scheme₀ authority₀ (File directory₀ file₀) query₀) headers₀ ->
                [ scheme₁, headers₁, authority₁ ]
            <>  directory₁
            <>  [ file₁, query₁ ]
          where
            scheme₁ = case scheme₀ of
                HTTP  -> TInt 0
                HTTPS -> TInt 1

            authority₁ = TString authority₀

            headers₁ = case headers₀ of
                Just h  -> encode h
                Nothing -> TNull

            directory₁ = map TString (reverse directory₀)

            query₁ = case query₀ of
                Just q  -> TString q
                Nothing -> TNull

            file₁ = TString file₀

        Path filePrefix₀ (File directory₀ file₀) ->
            filePrefix₁ : directory₁ <> [ file₁ ]
          where
            directory₁ = map TString (reverse directory₀)

            filePrefix₁ = case filePrefix₀ of
                Absolute -> TInt 2
                Here     -> TInt 3
                Parent   -> TInt 4
                Home     -> TInt 5

            file₁ = TString file₀

        Env x ->
            [ TInt 6, TString x ]

        Missing ->
            [ TInt 7 ]
```

### `let` expressions

A `let` binder is represented by a sequence of three elements: name, type annotation (`null` if absent) and bound expression. Adjacent `let` expressions are "flattened" and encoded in a single array, concatenating the immediately nested binders:

    encode(A₀) = A₁   encode(a₀) = a₁   encode(b₀) = b₁   ...   encode(z₀) = z₁
    ─────────────────────────────────────────────────────────────────────────────────────────────
    encode(let x : A₀ = a₀ in let y = b₀ ... in z₀) = [ 25, "x", A₁, a₁, "y", null, b₁, ..., z₁ ]


```haskell
encode expression@(Let _ _ _ _) = loop id expression
  where
    loop difference (Let x (Just _A₀) a₀ c) =
        loop (difference . ([ TString x, _A₁, a₁ ] <>)) c
      where
        _A₁ = encode _A₀

        a₁ = encode a₀

    loop difference (Let y Nothing b₀ c) =
        loop (difference . ([ TString y, TNull, b₁ ] <>)) c
      where
        b₁ = encode b₀

    loop difference z₀ = TList (TInt 25 : difference [ z₁ ])
      where
        z₁ = encode z₀
```

### Type annotations


    encode(t₀) = t₁   encode(T₀) = T₁
    ─────────────────────────────────
    encode(t₀ : T₀) = [ 26, t₁, T₁ ]


```haskell
encode (Annotation t₀ _T₀) = TList [ TInt 26, t₁, _T₁ ]
  where
    t₁ = encode t₀

    _T₁ = encode _T₀
```

### `with` expressions

A `with` expression is encoded as a list beginning with `29`, followed by the
subject of the expression, the path, and the value for the update.

Special path components are encoded as ints.
* `?` is encoded as `0`


    encode(e₀ with k₀.ks₀… = v₀) = [ 29, e₁, [ k₁, ks₁… ], v₁ ]
    ────────────────────────────────────────────────────────────────
    encode(e₀ with ?.k₀.ks₀… = v₀) = [ 29, e₁, [ 0, k₁, ks₁… ], v₁ ]


    encode(e₀) = e₁   encode(v₀) = v₁
    ──────────────────────────────────────────────
    encode(e₀ with ? = v₀) = [ 29, e₁, [ 0 ], v₁ ]


All other path components are encoded as strings.


    encode(e₀ with k₀.ks₀… = v₀) = [ 29, e₁, [ k₁, ks₁… ], v₁ ]
    ────────────────────────────────────────────────────────────────  ; k ≠ ?
    encode(e₀ with k.k₀.ks₀… = v₀) = [ 29, e₁, [ k, k₁, ks₁… ], v₁ ]


    encode(e₀) = e₁   encode(v₀) = v₁
    ──────────────────────────────────────────────  ; k ≠ ?
    encode(e₀ with k = v₀) = [ 29, e₁, [ k ], v₁ ]


```haskell
encode (With e₀ (k₀ :| ks₀) v₀) = TList [ TInt 29, e₁, TList (k₁ : ks₁), v₁ ]
  where
    e₁ = encode e₀
    v₁ = encode v₀

    k₁ = encodeKey k₀
    ks₁ = map encodeKey ks₀

    encodeKey DescendOptional = TInt 0
    encodeKey (Label k)       = TString k
```

### `Date` / `Time` / `TimeZone`


    ─────────────────────────────────────────
    encode(YYYY-MM-DD) = [ 30, YYYY, MM, DD ]


```haskell
encode (DateLiteral d) = TList [ TInt 30, TInt (fromInteger _YYYY), TInt _MM, TInt _DD ]
  where
    (_YYYY, _MM, _DD) = Time.toGregorian d
```


    ss = m*10^e
    ─────────────────────────────────────────
    encode(hh:mm:ss) = [ 31, hh, mm, m*10^e ]


```haskell
encode (TimeLiteral (Time.TimeOfDay hh mm ss) precision) =
    TList [ TInt 31, TInt hh, TInt mm, TTagged 4 (TList [ TInt e, m ]) ]
  where
    e = negate precision

    mantissa :: Integer
    mantissa = truncate (ss * 10^precision)

    m   |   fromIntegral (minBound :: Int) <= mantissa
        &&  mantissa <= fromIntegral (maxBound :: Int) =
            TInt (fromInteger mantissa)
        | otherwise =
            TInteger mantissa
```


    ─────────────────────────────────────
    encode(+HH:MM) = [ 32, True, HH, MM ]


    ──────────────────────────────────────
    encode(-HH:MM) = [ 32, False, HH, MM ]


```haskell
encode (TimeZoneLiteral (Time.TimeZone minutes _ _)) =
    TList [ TInt 32, TBool sign, TInt _HH, TInt _MM ]
  where
    sign = 0 <= minutes

    (_HH, _MM) = abs minutes `divMod` 60
```


## Decoding judgment


You can decode a Dhall expression using the following judgment:

    decode(cbor) = dhall

... where:

* `cbor` (the input) is a CBOR expression
* `dhall` (the output) is a Dhall expression

### CBOR Tags

Dhall does not currently recognize any CBOR tags that add semantics to data
items.  However, [tag 55799 is defined][self-describe-cbor] to mean that the
following data item has exactly the same semantics it would have without the
tag.  Therefore this tag should be allowed by decoders, and ignored:


    decode(x) = e
    ───────────────────────────────────────────
    decode(CBORTag 55799 x) = e


### Built-in constants

A naked CBOR string represents a built-in identifier.  Decode the string as
a built-in identifier if it matches any of the following strings:

    ───────────────────────────────────────────
    decode("Natural/build") = Natural/build


    ─────────────────────────────────────────
    decode("Natural/fold") = Natural/fold


    ─────────────────────────────────────────────
    decode("Natural/isZero") = Natural/isZero


    ─────────────────────────────────────────
    decode("Natural/even") = Natural/even


    ───────────────────────────────────────
    decode("Natural/odd") = Natural/odd


    ───────────────────────────────────────────────────
    decode("Natural/toInteger") = Natural/toInteger


    ─────────────────────────────────────────
    decode("Natural/show") = Natural/show


    ─────────────────────────────────────────────────
    decode("Integer/toDouble") = Integer/toDouble


    ─────────────────────────────────────────
    decode("Integer/show") = Integer/show


    ─────────────────────────────────────────────
    decode("Integer/negate") = Integer/negate


    ───────────────────────────────────────────
    decode("Integer/clamp") = Integer/clamp


    ───────────────────────────────────────
    decode("Double/show") = Double/show


    ─────────────────────────────────────
    decode("List/build") = List/build


    ───────────────────────────────────
    decode("List/fold") = List/fold


    ───────────────────────────────────────
    decode("List/length") = List/length


    ───────────────────────────────────
    decode("List/head") = List/head


    ───────────────────────────────────
    decode("List/last") = List/last


    ─────────────────────────────────────────
    decode("List/indexed") = List/indexed


    ─────────────────────────────────────────
    decode("List/reverse") = List/reverse


    ─────────────────────────────────────
    decode("Text/replace") = Text/replace


    ───────────────────────────────
    decode("Text/show") = Text/show


    ───────────────────────────────
    decode("Date/show") = Date/show


    ───────────────────────────────
    decode("Time/show") = Time/show


    ───────────────────────────────────────
    decode("TimeZone/show") = TimeZone/show


    ─────────────────────────
    decode("Bool") = Bool


    ─────────────────────────────────
    decode("Optional") = Optional


    ─────────────────────────
    decode("None") = None


    ───────────────────────────────
    decode("Natural") = Natural


    ───────────────────────────────
    decode("Integer") = Integer


    ─────────────────────────────
    decode("Double") = Double


    ─────────────────────────
    decode("Text") = Text


    ─────────────────────────
    decode("Bytes") = Bytes


    ─────────────────────────
    decode("List") = List


    ─────────────────────────
    decode("Date") = Date


    ─────────────────────────
    decode("Time") = Time


    ─────────────────────────────
    decode("TimeZone") = TimeZone


    ─────────────────────────
    decode("Type") = Type


    ─────────────────────────
    decode("Kind") = Kind


    ─────────────────────────
    decode("Sort") = Sort


Otherwise, there is a decoding error.

### Variables

A CBOR array beginning with a string indicates a variable.  The array should
only have two elements, the first of which is a string and the second of which
is the variable index, which can be either a compact integer or a bignum:


    ────────────────────────────
    decode([ "x", n ]) = x@n


    ─────────────────────────────
    decode([ "x", nn ]) = x@n


A decoder MUST accept an integer that is not encoded using the most compact
representation.  For example, a naked unsigned bignum storing `0` is valid, even
though the `0` could have been stored in a single byte as as a compact unsigned
integer instead.

A decoder MUST reject a variable named "_", which MUST instead be represented
by its integer index according to the following decoding rules.

Only variables named `_` encode to a naked CBOR integer, so decoding turns a
naked CBOR integer back into a variable named `_`:


    ───────────────────
    decode(n) = _@n


    ────────────────────
    decode(nn) = _@n


As before, the encoded integer need not be stored in the most compact
representation.

### Function application

Decode a CBOR array beginning with `0` as function application, where the second
element is the function and the remaining elements are the function arguments:


    decode(f₁) = f₀   decode(a₁) = a₀   decode(b₁) = b₀   …
    ───────────────────────────────────────────────────────────────────
    decode([ 0, f₁, a₁, b₁, … ]) = f₀ a₀ b₀ …


A decoder MUST require at least 1 function argument.  In other words, a decode
MUST reject a CBOR array of of the form `[ 0, f₁ ]`.

### Functions

Decode a CBOR array beginning with a `1` as a λ-expression.  If the array has
three elements then the bound variable is named `_`:


    decode(A₁) = A₀   decode(b₁) = b₀
    ──────────────────────────────────────────
    decode([ 1, A₁, b₁ ]) = λ(_ : A₀) → b₀


... otherwise if the array has four elements then the name of the bound variable
is included in the array:


    decode(A₁) = A₀   decode(b₁) = b₀
    ───────────────────────────────────────────────  ; x ≠ "_"
    decode([ 1, "x", A₁, b₁ ]) = λ(x : A₀) → b₀


A decoder MUST reject the latter form if the bound variable is named `_`

Decode a CBOR array beginning with a `2` as a ∀-expression.  If the array has
three elements then the bound variable is named `_`:


    decode(A₁) = A₀   decode(B₁) = B₀
    ──────────────────────────────────────────
    decode([ 2, A₁, B₁ ]) = ∀(_ : A₀) → B₀


... otherwise if the array has four elements then the name of the bound variable
is included in the array:


    decode(A₁) = A₀   decode(B₁) = B₀
    ───────────────────────────────────────────────  ; x ≠ "_"
    decode([ 2, "x", A₁, B₁ ]) = ∀(x : A₀) → B₀


A decoder MUST reject the latter form if the bound variable is named `_`

### Operators

Decode a CBOR array beginning with a `3` as an operator expression:


    decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────
    decode([ 3, 0, l₁, r₁ ]) = l₀ || r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────
    decode([ 3, 1, l₁, r₁ ]) = l₀ && r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────
    decode([ 3, 2, l₁, r₁ ]) = l₀ == r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────
    decode([ 3, 3, l₁, r₁ ]) = l₀ != r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ──────────────────────────────────
    decode([ 3, 4, l₁, r₁ ]) = l₀ + r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ──────────────────────────────────
    decode([ 3, 5, l₁, r₁ ]) = l₀ * r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────
    decode([ 3, 6, l₁, r₁ ]) = l₀ ++ r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ──────────────────────────────────
    decode([ 3, 7, l₁, r₁ ]) = l₀ # r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ──────────────────────────────────
    decode([ 3, 8, l₁, r₁ ]) = l₀ ∧ r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ──────────────────────────────────
    decode([ 3, 9, l₁, r₁ ]) = l₀ ⫽ r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────
    decode([ 3, 10, l₁, r₁ ]) = l₀ ⩓ r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────
    decode([ 3, 11, l₁, r₁ ]) = l₀ ? r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ─────────────────────────────────────
    decode([ 3, 12, l₁, r₁ ]) = l₀ === r₀


    decode(l₁) = l₀   decode(r₁) = r₀
    ────────────────────────────────────
    decode([ 3, 13, l₁, r₁ ]) = l₀ :: r₀


### `List`

Decode a CBOR array beginning with a `4` or a `28` as a `List` literal

If the list is empty, then the type MUST be non-`null`:


    decode(T₁) = T₀
    ────────────────────────────────────
    decode([ 4, T₁ ]) = [] : List T₀


    decode(T₁) = T₀
    ────────────────────────────
    decode([ 28, T₁ ]) = [] : T₀


If the list is non-empty then the type MUST be `null`:


    decode(a₁) = a₀   decode(b₁) = b₀
    ──────────────────────────────────────────────────
    decode([ 4, null, a₁, b₁, … ]) = [ a₀, b₀, … ]


### `Some`

Decode a CBOR array beginning with a `5` as a `Some` expression:


    decode(t₁) = t₀
    ─────────────────────────────────────
    decode([ 5, null, t₁ ]) = Some t₀


### `merge` expressions

Decode a CBOR array beginning with a `6` as a `merge` expression:


    decode(t₁) = t₀   decode(u₁) = u₀
    ─────────────────────────────────────────
    decode([ 6, t₁, u₁ ]) = merge t₀ u₀


    decode(t₁) = t₀   decode(u₁) = u₀   decode(T₁) = T₀
    ───────────────────────────────────────────────────────────────
    decode([ 6, t₁, u₁, T₁ ]) = merge t₀ u₀ : T₀


### `toMap` expressions

Decode a CBOR array beginning with a `27` as a `toMap` expression:


    decode(t₁) = t₀
    ─────────────────────────────
    decode([ 27, t₁ ]) = toMap t₀


    decode(t₁) = t₀   decode(T₁) = T₀
    ──────────────────────────────────────
    decode([ 27, t₁, T₁ ]) = toMap t₀ : T₀


### `showConstructor` expressions

Decode a CBOR array beginning with a `34` as a `showConstructor` expression:


    decode(t₁) = t₀
    ─────────────────────────────
    decode([ 34, t₁ ]) = showConstructor t₀


### Records

Decode a CBOR array beginning with a `7` as a record type:


    decode(T₁) = T₀   …
    ──────────────────────────────────────────────────
    decode([ 7, { "x" = T₁, … } ]) = { x : T₀, … }


Decode a CBOR array beginning with a `8` as a record literal:


    decode(t₁) = t₀   …
    ──────────────────────────────────────────────────
    decode([ 8, { "x" = t₁, … } ]) = { x = t₀, … }


Decode a CBOR array beginning with a `9` as a field access:


    decode(t₁) = t₀   …
    ─────────────────────────────────
    decode([ 9, t₁, "x" ]) = t₀.x


Decode a CBOR array beginning with a `10` as a record projection:


    decode(t₁) = t₀   …
    ────────────────────────────────────────────────────
    decode([ 10, t₁, "x", "y", … ]) = t₀.{ x, y, … }


    decode(t₁) = t₀   decode(T₁) = T₀
    ────────────────────────────────
    decode([ 10, t₁, [ T₁ ] ]) = t₀.(T₀)


A decoder MUST NOT attempt to enforce uniqueness of keys.  That is the
responsibility of the type-checking phase.

### Unions

Decode a CBOR array beginning with a `11` as a union type:


    decode(T₁) = T₀   …
    ────────────────────────────────────────────────────────────────
    decode([ 11, { "x" = T₁, "y" = null, … } ]) = < x : T₀ | y | … >


A decoder MUST NOT attempt to enforce uniqueness of keys.  That is the
responsibility of the type-checking phase.

### `Bool`

Decode CBOR boolean values to Dhall boolean values:


    ───────────────────────
    decode(True) = True


    ─────────────────────────
    decode(False) = False


Decode a CBOR array beginning with a `14` as an `if` expression:


    decode(t₁) = t₀   decode(l₁) = l₀   decode(r₁) = r₀
    ───────────────────────────────────────────────────────────────
    decode([ 14, t₁, l₁, r₁ ]) = if t₀ then l₀ else r₀


### `Natural`

Decode a CBOR array beginning with a `15` as a `Natural` literal:


    ─────────────────────────
    decode([ 15, n ]) = n


    ──────────────────────────
    decode([ 15, nn ]) = n


A decoder MUST accept an integer that is not encoded using the most compact
representation.

### `Integer`

Decode a CBOR array beginning with a `16` as an `Integer` literal:


    ────────────────────────────
    decode([ 16, -nn ]) = ±n


    ───────────────────────────
    decode([ 16, -n ]) = ±n


    ──────────────────────────
    decode([ 16, n ]) = ±n


    ───────────────────────────
    decode([ 16, nn ]) = ±n


A decoder MUST accept an integer that is not encoded using the most compact
representation.

### `Double`

Decode a CBOR double, single, or half and convert to a `Double` literal:


    ─────────────────────────────
    decode(n.n) = n.n


    ─────────────────────────────
    decode(n.n_s) = n.n


    ─────────────────────────────
    decode(n.n_h) = n.n


### `Text`

Decode a CBOR array beginning with a `18` as a `Text` literal:


    decode(b₁) = b₀   decode(d₁) = d₀   …   decode(y₁) = y₀
    ───────────────────────────────────────────────────────────────────────────────────
    decode([ 18, "a", b₁, "c", d₁, "e", …, "x", y₁, "z" ]) = "a${b₀}c${d}e…x${y₀}z"


### `Bytes`

Decode a CBOR array beginning with a `33` as a `Bytes` literal:


    base16encode(bytes) = 0x"0123456789abcdef"
    ───────────────────────────────────────────────
    decode([ 33, b"bytes" ]) = 0x"0123456789abcdef"


The `base16encode` judgement stands in for the base-16 encoding algorithm
specified in
[RFC4648 - Section 8](https://tools.ietf.org/html/rfc4648#section-8), treated
as a pure function from a byte array to text.

### `assert`

Decode a CBOR array beginning with a `19` as an `assert`:


    decode(T₁) = T₀
    ────────────────────────────────
    decode([ 19, T₁ ]) = assert : T₀


### Imports

Decode a CBOR array beginning with a `24` as an import.

The decoding rules are the exact opposite of the encoding rules:


    ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 0, null, "authority", "path₀", "path₁", …, "file", "query" ]) = http://authority/path₀/path₁/…/file?query


    ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 0, null, "authority", "path₀", "path₁", …, "file", null ]) = http://authority/path₀/path₁/…/file


    ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 1, null, "authority", "path₀", "path₁", …, "file", "query" ]) = https://authority/path₀/path₁/…/file?query


    ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 1, null, "authority", "path₀", "path₁", …, "file", null ]) = https://authority/path₀/path₁/…/file


    ─────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 2, "path₀", "path₁", …, "file" ]) = /path₀/path₁/…/file


    ──────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 3, "path₀", "path₁", …, "file" ]) = ./path₀/path₁/…/file


    ───────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 4, "path₀", "path₁", …, "file" ]) = ../path₀/path₁/…/file


    ──────────────────────────────────────────────────────────────────────────────
    decode([ 24, null, 0, 5, "path₀", "path₁", …, "file" ]) = ~/path₀/path₁/…/file


    ───────────────────────────────────────
    decode([ 24, null, 0, 6, "x" ]) = env:x


    ────────────────────────────────────
    decode([ 24, null, 0, 7 ]) = missing


    decode([ 24, x, 0, xs… ]) = import
    ──────────────────────────────────────────
    decode([ 24, x, 1, xs… ]) = import as Text


    decode([ 24, x, 0, xs… ]) = import
    ───────────────────────────────────────────
    decode([ 24, x, 3, xs… ]) = import as Bytes


    decode([ 24, x, 0, xs… ]) = import
    ──────────────────────────────────────────────
    decode([ 24, x, 2, xs… ]) = import as Location


    decode([ 24, null, x, xs… ]) = import
    base16encode(rawHash) = base16Hash
    ─────────────────────────────────────────────────────────────────────────────
    decode([ 24, b"\x12\x20rawHash", x, xs… ]) = import sha256:base16Hash


    decode(headers₀) = headers₁
    decode([ 24, x, y, 0, null, xs… ]) = http://authority directory file
    ─────────────────────────────────────────────────────────────────────────────────────
    decode([ 24, x, y, 0, headers₀, xs… ]) = http://authority directory file using headers₁


    decode(headers₀) = headers₁
    decode([ 24, x, y, 1, null, xs… ]) = https://authority directory file
    ──────────────────────────────────────────────────────────────────────────────────────
    decode([ 24, x, y, 1, headers₀, xs… ]) = https://authority directory file using headers₁


### `let` expressions

Decode a CBOR array beginning with a `25` as a `let` expression:


    decode(A₁) = A₀   decode(a₁) = a₀   decode(b₁) = b₀   ...   decode(z₁) = z₀
    ──────────────────────────────────────────────────────────────────────────────────────────
    decode([ 25, "x", A₁, a₁, "y", null, b₁, ..., z₁ ]) = let x : A₀ = a₀ let y = b₀ ... in z₀


### Type annotations


    decode(t₁) = t₀   decode(T₁) = T₀
    ─────────────────────────────────
    decode([ 26, t₁, T₁ ]) = t₀ : T₀


### `with` expressions


    decode(e₁) = e₀   decode(v₁) = v₀
    ─────────────────────────────────────────────────────
    decode([29, e₁, [ k, ks… ], v₁]) = e₀ with k.ks… = v₀


### `Date` / `Time` / `TimeZone`


    ───────────────────────────────────────
    decode([30, YYYY, MM, DD]) = YYYY-MM-DD


    m*10^e = ss
    ───────────────────────────────────────
    decode([31, hh, mm, m*10^e]) = hh:mm:ss


    ───────────────────────────────────
    decode([32, true, HH, MM]) = +HH:MM


    ────────────────────────────────────
    decode([32, false, HH, MM]) = -HH:MM


[self-describe-cbor]: https://tools.ietf.org/html/rfc7049#section-2.4.5
[multihash]: https://github.com/multiformats/multihash
