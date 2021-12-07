# Valuable Value (vv)

**status: approaching stabilization upon finishing a reference implementation, but currently still tweaking details**

Specification of a set of values, orderings on them, and some encodings for them, suitable for data interchange.

## Motivation

The valuable value specification targets similar use cases as [json](https://www.json.org/json-en.html), [toml](https://toml.io/en/) and [yaml](https://yaml.org/). These (and similar) data formats approach design from a syntactic perspective; their specifications usually detail which byte sequences a parser should accept or reject. This frequently leads to ambiguities around the behavior of edge cases, such as extremely long integer literals or the joys of [NaN](https://en.wikipedia.org/wiki/NaN) values. The result are incompatibilities between different implementations. Any specification that builds on these formats inherits those ambiguities.

To approach this problem, the valuable value specification provides a well-defined data type, and then specifies exactly how any instance of that data type can be encoded. A welcome side effect from decoupling logical values and syntax is that multiple encodings with distinct goals and properties can be defined. This specification provides a human-readable encoding, a compact machine-friendly encoding, as well as a canonic subset of the compact encoding suitable for deterministic [hashing](https://en.wikipedia.org/wiki/Hash_function).

## Values

A valuable value (just *value* from now on) is any of the following:

- `nil`: A value that carries [no further information](https://en.wikipedia.org/wiki/Unit_type).
- `boolean`: Either `true` or `false`.
- `float`: An [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) double precision float, with the only difference that there is only one kind of NaN.
- `int`: An integer between `-(2^63)` and `(2^63) - 1` (inclusive).
- `array`: An ordered sequence of up to `(2^63) - 1` values.
- `map`: An unordered collection of up to `(2^63) - 1` pairs (`entries`) of values, where the first values of all entries are pairwise distinct. The first value of an entry is called a *key*, the second value is the *corresponding value*.

The choice of values a self-describing format provides is to some degree arbitrary. For vv, decisions between different approaches have often been decided based on machine-friendliness. Fixed-width integers are much easier to handle than arbitrary precision integers. Floats are easier than true rationals. Maximum collection sizes enable implementations to precisely follow the spec rather than introducing arbitrary limits that would inevitably differ between distinct implementations.

## Relations

This section defines some binary relations that end up being useful when working with valuable values. The equality relation and the canonic linear order are necessary for defining some encoding details later on, the subvalue relation is included for use by other specifications that require a more intuitive way of comparing values than the canonic linear order provides.

### Equality

The equality relation on values is an [equivalence relation](https://en.wikipedia.org/wiki/Equivalence_relation) (not a given whenever floating-point numbers are involved).

- `nil` is equal to `nil` and no other value
- `false` is equal to `false` and no other value
- `true` is equal to `true` and no other value
- two floats are equal if and only if both are NaN or their bit representation is equal
- two ints are equal if and only if their bit representation is equal
- two arrays are equal if and only if they have the same length and they are element-wise equal
- two maps are equal if and only if every key in the first map is also a key in the second map and vice versa, and both maps associate equal values with equal keys

### Subvalues

The subvalue relation is a [(reflexive) partial order](https://en.wikipedia.org/wiki/Partially_ordered_set#Formal_definition) with intuitive [greatest lower bounds and least upper bounds](https://en.wikipedia.org/wiki/Infimum_and_supremum) (whenever they exist).

We write `v <= w` to denote that the value `v` is a subvalue of the value `w`.

For all values `v` we have `v <= v`. Furthermore, two values of the same kind might be in relation:

- `true` is greater than `false`.
- Floats are ordered according to the IEEE 754-2008 totalOrder predicate, treating NaN as a positive, quiet NaN whose payload bits are all set to one: negative infinity < finite numbers, in ascending order, with negative zero before positive zero < positive infinity < NaN.
- Ints are sorted numerically (e.g. `-1 < 0 < 1`).
- Arrays require a recursive definition: If one of the arrays contains strictly fewer elements than the other, it is treated as if it was right-padded to the length of the other array with values that are a subvalue of any actual valuable value. One of these arrays is a subvalue of the other if at each index the value in the one is a subvalue of the value in the other.
- Maps require a recursive definition: For any key contained in one map but not the other, the other map is treated as if it mapped that key to a value that is a subvalue of any actual valuable value. One of these maps is a subvalue of the other if it maps every key to a subvalue of the value to which the other map maps that same key.

There is no defined order between values of different kinds, e.g., neither `2 <= true` nor `true <= 2`. In particular, floats and integers are incomparable.

### Canonic Linear Order

The canonic linear order is a [linear order](https://en.wikipedia.org/wiki/Total_order) and a [linear extension](https://en.wikipedia.org/wiki/Linear_extension) of the subvalue relation.

The order is the transitive closure of the following base definitions:

- `nil` is less than any other value
- booleans are greater than `nil`
- floats are greater than booleans
- ints are greater than floats
- arrays are greater than ints
- maps are greater than arrays
- `true` is greater than `false`
- floats are ordered according to the IEEE 754-2008 totalOrder predicate, treating NaN as a positive, quiet NaN whose payload bits are all set to one: negative infinity < finite numbers, in ascending order, with negative zero before positive zero < positive infinity < NaN
- ints are sorted numerically (e.g. `-1 < 0 < 1`)
- arrays are sorted [lexicographically](https://en.wikipedia.org/wiki/Lexicographic_order)
- maps are a bit more complicated, because the standard approach of using lexicographic order after sorting the entries is incompatible with the subvalue relation
  - the empty map is less than any other map
  - when comparing two nonempty maps `m1` and `m2`, let `k1` and `k2` be the least keys of `m1` and `m2` respectively, and let `v1` and `v2` be the corresponding values
    - if `k1 < k2`, then `m2 < m1`
    - if `k2 < k1`, then `m1 < m2`
    - if `k1 = k2` and `v1 < v2`, then `m1 < m2`
    - if `k1 = k2` and `v2 < v1`, then `m2 < m1`
    - if `k1 = k2` and `v1 = v2`, then compare the maps obtained by removing the entries for `k1` and `k2` from `m1` and `m2` respectively

## Encodings

There are different encodings of values, suitable for different use cases:

- The *human-readable encoding* is inefficient but nice for humans to work with.
- The *compact encoding* is space-efficient and easy to parse for machines but not for humans
- The *canonic encoding* is a subset of the compact encoding that guarantees a one-to-one mapping between codes and values.

It might be convenient to define further encodings for particular use cases. Any [injective function](https://en.wikipedia.org/wiki/Injective_function) from the valuable values to the set of byte sequences constitutes a proper encoding.

In the following, a *string* refers to an array whose entries are integers between `0` and `255`, and a *set* refers to a map that maps all its keys to `nil`. Both of these get more compact and easy to write encodings.

### Human-Readable Encoding

The human-readable encoding is intended to be read, created and edited by humans directly. The recommended file extension is `.vv`. Its syntax utilizes elements from the [common syntax specification](https://github.com/AljoschaMeyer/common_syntax).

*Whitespace* is defined as [common syntax whitespace](https://github.com/AljoschaMeyer/common_syntax#whitespace). A valid  human-readable value code consists of any amount of whitespace, followed by the encoding of a value as described next, followed by any amount of whitespace.

#### Nil

`nil` is encoded as the utf-8 string `nil`.

#### Booleans

`true` is encoded as the utf-8 string `true`.

`false` is encoded as the utf-8 string `false`.

#### Floats

A float is encoded as a [common syntax floating-point literal](https://github.com/AljoschaMeyer/common_syntax#floating-point-literal).

#### Ints

An int is encoded as a [common syntax integer literal](https://github.com/AljoschaMeyer/common_syntax#integer-literal) between `-(2^63)` and `(2^63) - 1`.

#### Arrays

An array is encoded as a comma-separated list of the encodings of the contained values, enclosed between brackets `[` and `]`. An optional trailing comma before the closing bracket is allowed. Any amount of whitespace can be placed between brackets, contained values, and commas. The list may contain at most `(2^63) - 1` entries.

Strings (arrays whose entries are ints between `0` and `255`) can also be encoded as [common syntax byte string literals](https://github.com/AljoschaMeyer/common_syntax#byte-string) or as [common syntax UTF-8 string literals](https://github.com/AljoschaMeyer/common_syntax#utf-8-string).

#### Maps

A map is encoded as a comma-separated list of the contained pairs (see below), enclosed between braces `{` and `}`. An optional trailing comma before the closing brace is allowed. Any amount of whitespace can be placed between braces, contained values, and commas. The list may contain at most `(2^63) - 1` entries.

An entry is encoded as the encoding of the key, followed by any amount of whitespace, followed by a `:` followed by any amount of whitespace, followed by the encoding of the value.

When decoding multiple entries with identical keys, a later entry replaces the earlier entry in the map.

#### Sets

A set (a map which maps all keys to `nil`) can also be encoded as a comma-separated list of the encodings of the keys, enclosed between `@{` and `}`. An optional trailing comma before the closing brace is allowed. Any amount of whitespace can be placed between braces, contained values, and commas. The list may contain at most `(2^63) - 1` entries.

When decoding, duplicate values are allowed and are simply discarded.

### Compact Encoding

The compact encoding uses techniques similar to [cbor](https://tools.ietf.org/html/rfc7049) to make it space-efficient and easy to parse. There is no concept of whitespace (and thus comments) in the compact encoding. The recommended file extension is `.cvv`.

Values are encoded as a single byte that tags the kind of value, sometimes followed by more, actual data. The first three bits of all tags indicate the kind of the value. The remaining five bits carry additional information about the following data.

Some values have multiple valid encodings. Implementations are strongly encouraged to always use the shortest possible encoding, but reading an overlong code is *not* a decoding error.

#### Nil

`nil` is encoded as the tag `0b000_00000`, followed by no additional data.

#### Booleans

`false` is encoded as the tag `0b001_00000`, followed by no additional data.

`true` is encoded as the tag `0b001_00001`, followed by no additional data.

#### Floats

Floats are encoded as the tag `0b010_00000`, followed by the eight bytes of the float (sign, exponent, fraction in that order). NaN can be encoded as any valid IEEE 754 NaN (arbitrary sign, exponent all ones, remaining bytes arbitrary but non-zero).

#### Ints

Ints are encoded as the tag `0b011_xxxxx`, where the least significant five bits and the following bytes are determined as follows:

- for least significant bits strictly less than `0b11100`, the bits themselves represent the encoded int (in the range from zero to 27), no more bytes follow the tag
- for least significant bits `0b11100`, the tag is followed by a single byte, which encodes the int as two's complement (ranging from `-(2^7)` to `(2^7) - 1`)
- for least significant bits `0b11101`, the tag is followed by two bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^15)` to `(2^15) - 1`)
- for least significant bits `0b11110`, the tag is followed by four bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^31)` to `(2^31) - 1`)
- for least significant bits `0b11111`, the tag is followed by eight bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^63)` to `(2^63) - 1`)

#### Strings

Strings are encoded as the tag `0b100_xxxxx`, where the least significant five bits and the following bytes are determined as follows:

- for least significant bits strictly less than `0b11100`, the bits themselves represent the length of the string, the tag is followed by that many bytes
- for least significant bits `0b11100`, the tag is followed by a single byte, which encodes the length of the string, followed by that many bytes
- for least significant bits `0b11101`, the tag is followed by two bytes in big-endian order, which encode the length of the string, followed by that many bytes
- for least significant bits `0b11110`, the tag is followed by four bytes in big-endian order, which encode the length of the string, followed by that many bytes
- for least significant bits `0b11111`, the tag is followed by eight bytes in big-endian order, which encode the length of the string, followed by that many bytes

When decoding, reading a length of more than `2^63 - 1` is an *error*.

#### Arrays

Arrays are encoded just like strings, with the following differences:

- they use the tags `0b101_xxxxx`
- the length is not given in bytes but in items
- after the length, the code is followed by the compact encodings of all items in order

#### Sets

Sets are encoded just like arrays, with the following differences:

- they use the tags `0b110_xxxxx`
- when decoding, duplicate values are allowed but are simply discarded

#### Maps

Maps are encoded just like arrays, with the following differences:

- they use the tags `0b111_xxxxx`
- the length is not given in items but in entries
- after the length, the code is followed by the encodings of all entries, where an entry is encoded as a compact encoding of its key followed by a compact encoding of its value
- when decoding multiple entries with identical keys, a later entry replaces the earlier entry in the map

### Canonic Encoding

A canonic encoding is one where there is a one-to-one correspondence between values and codes. The valuable value canonic encoding is a subset of the compact encoding, obtained through the following restrictions:

- ints, arrays and maps must use the shortest possible encodings for their length/size
- NaN must be encoded as the tag for a float followed by 64 1-bits
- strings are always encoded as regular arrays
- sets are always encoded as regular maps
- map entries must be sorted ascendingly by their keys, according to the canonic linear order, each key occuring at most once
