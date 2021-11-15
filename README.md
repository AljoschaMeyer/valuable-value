# Valuable Value (vv)

Specification of a set of values, orderings on them, and some encodings for them, suitable for data interchange.

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

This section defines some binary relations that end up being useful when working with values.

### Equality

The equality relation on values is an [equivalence relation](https://en.wikipedia.org/wiki/Equivalence_relation).

- `nil` is equal to `nil` and no other value
- `false` is equal to `false` and no other value
- `true` is equal to `true` and no other value
- two floats are equal if and only if both are NaN or their bit representation is equal
- two ints are equal if and only if their bit representation is equal
- two arrays are equal if and only if they have the same length and they are element-wise equal
- two maps are equal if and only if every key in the first map is also a key in the second map and vice versa, and both maps associate equal values with equal keys

### Linear Order

Defining any [linear order](https://en.wikipedia.org/wiki/Total_order) over the valuable values involves to some degree arbitrary choices. This order compares values by their kind first, and then within the same kind. This leads to a simple definition and implementation, though some consequences might seem odd at first glance: for example, the float `1.0` is less than the int `0`, because any float is considered less then any int.

Staying compatible with the preceeding definition of equality, negative and positive float zero are considered distinct values, with `-0.0 < 0.0`, unlike the default order defined in the IEEE 754 standard.

The order is the transitive closure of the following base definitions:

- `nil` is less than any other value
- booleans are greater than `nil`
- floats are greater than booleans
- ints are greater than floats
- arrays are greater than strings
- maps are greater than sets
- `true` is greater than `false`
- floats are ordered according to the IEEE 754-2008 totalOrder predicate, treating NaN as a positive, quiet NaN whose payload bits are all set to one: negative infinity < finite numbers, in ascending order, with negative zero before positive zero < positive infinity < NaN
- ints are sorted numerically (e.g. `-1 < 0 < 1`)
- arrays are sorted [lexicographically](https://en.wikipedia.org/wiki/Lexicographic_order)
- maps are sorted amongst themselves as if they were arrays containing the entries in ascending order of their keys, each entry being a two-element array whose first element is the key and whose second element is the corresponding value

### A Meaningful Partial Order

The linear order make some arbitrary decisions to achieve linearity, resulting in fairly meaningless [greatest lower bounds and least upper bounds](https://en.wikipedia.org/wiki/Infimum_and_supremum). The following [partial order](https://en.wikipedia.org/wiki/Partially_ordered_set#Formal_definition) makes more meaningful choices, while still being compatible with the linear order: if `v < w` in the partial order, then also `v < w` in the linear order.

There is no defined order between values of different kinds. Within a kind, the ordering is the same as the linear order for `nil`, booleans, floats, and ints. Arrays and maps are ordered as follows:

- If one of the arrays is strictly smaller than the other, it is conceptually right-padded with values that are strictly less than any real value. The arrays are equal if at each index they contain equal values. One is strictly less than the other if they are not equal and at each index the value in the one is less than or equal to the value in the other.
- For any key contained in one map but not the other, the other map is conceptually mapping that key to a value that is strictly less than any real value. The maps are then compared as if they were arrays containing the entries in ascending order of their keys, each entry being a two-element array whose first element is the key and whose second element is the corresponding value.

## Encodings

There are different encodings of values, suitable for different use cases:

- The *human-readable encoding* is inefficient but nice for humans to work with.
- The *compact encoding* is space-efficient and easy to parse for machines but not for humans.
- The *hybrid encoding* admits both human-readable and compact representations.
- The *canonic encoding* guarantees a one-to-one mapping between codes and values.

In the following, a *string* refers to an array whose entries are integers between `0` and `255`, and a *set* refers to a map that maps all its keys to `nil`. Both of these get more compact and easy to write encodings.

### Human-Readable Encoding

The human-readable encoding is intended to be read, created and edited by humans directly. The recommended file extension is `.vv`.

The bytes `0x09` (tab), `0x0a` (newline), `0x0d` (carriage return), and `0x20` (space) are considered whitespace. A sequence of valid utf-8 beginning with `#` and ending with either `0x0a` (newline) or the end of input is considered whitespace, called a (line) comment. A valid code consists of any amount of whitespace, followed by the encoding of a value as described next, followed by any amount of whitespace. Whitespace may only be inserted at the explicitly specified positions.

#### Nil

`nil` is encoded as the utf-8 string `nil`.

#### Booleans

`true` is encoded as the utf-8 string `true`.

`false` is encoded as the utf-8 string `false`.

#### Floats

Positive infinity is either encoded as the utf-8 string `Inf` or as the utf-8 string `+Inf`. NaN is encoded as `NaN`.

A negative float (including negative zero and negative infinity) is encoded as a `-` directly followed by an encoding of the same float but with the sign bit flipped, except that the encoding of the positive float may not begin with a `+`.

Non-negative floats are encoded as a decimal representation: one or more decimal ASCII digits (`0123456789`), followed by a `.`, followed by one or more decimal ASCII digits, optionally followed by an exponent: `e` or `E`, an optional sign (`+` or `-`), followed by one or more decimal ASCII digits.

A positive float encoding may be immediately preceded by a `+`, which is ignored when decoding.

Any character of a float encoding may be followed by an arbitrary number of underscores (`_`), which are ignored when decoding.

When decoding a float, the input may not be representable exactly. In these cases, use ["round to nearest, ties to even"](https://en.wikipedia.org/wiki/IEEE_754#Roundings_to_nearest) rounding mode. As a consequence, floats do not need to be encoded as their exact decimal representation, implementations may instead use a smaller decimal representation that still rounds to the correct float. Note that the rounding may result in an infinity, so it is possible to encode infinities as large decimal numbers (e.g. `9999.9e999999`) rather than `Inf`.

#### Ints

A negative int is encoded as a `-` directly followed by an encoding of the positive integer with the same absolute value (pretend for the purpose of this definition that `2^63` was valid value), except that the encoding of the positive integer may not begin with a `+`.

A positive int can be encoded in one of three ways: Either as a sequence of ASCII decimal digits, or as the utf-8 string `0x` followed by a sequence of ASCII hexadecimal digits (`0123456789abcdefABCDEF`), or as the utf-8 string `0b` followed by a sequence of binary digits (`01`).

A positive integer encoding may be immediately preceded by a `+`, which is ignored when decoding.

Any character of an int encoding may be followed by an arbitrary number of underscores, which are ignored when decoding.

When decoding, reading an integer outside the allowed range (between `-(2^63)` and `(2^63) - 1` inclusive) is an *error*. Those are not valid human-readably encoded values. Superfluous leading zeros are allowed.

#### Arrays

An array is encoded as a comma-separated list of the encodings of the contained values, enclosed between brackets `[` and `]`. An optional trailing comma before the closing bracket is allowed. Any amount of whitespace can be placed between brackets, contained values, and commas. The list may contain at most `(2^63) - 1` entries.

When decoding, reading an array of more than `(2^63) - 1` contained values is an *error*.

#### Strings

There are additional ways of encoding strings (reminder: a string is an array whose entries are integers between `0` and `255`).

A string can be encoded as a comma-separated (`,`) list of its bytes, enclosed between `@[` and `]`. The bytes are encoded just like `ints`, except that the numerical value must be between `0` and `255` inclusive. An optional trailing comma before the closing bracket is allowed. Any amount of whitespace can be placed between brackets, contained values, and commas.

A string can also be encoded as a hexadecimal representation of its numeric value: `@x[`, followed by an even number of hexadecimal digits, followed by `]`. Each hexadecimal digit may be preceded or succeeded by an arbitrary number of_, which are ignored when decoding.

A string can also be encoded as a binary representation of its numeric value: `@b[`, followed by a number of binary digits divisible by eight, followed by `]`. Each binary digit may be preceded or succeeded by an arbitrary number of_, which are ignored when decoding.

A string whose content is valid UTF-8 can be encoded as a `"`, followed by up to `(2^63) - 1` bytes worth of scalar encodings (see next paragraph), followed by another `"`.

Each [unicode scalar](http://www.unicode.org/glossary/#unicode_scalar_value) can either be encoded literally or through an escape sequence. The literal encoding can be used for all scalar values other than `"` and `\` and consists of the utf-8 encoding of the scalar value. Alternatively, any of the following escape sequences can be used:

- `\"` for the character `"`
- `\\` for the character `\`
- `\t` for the character `horizontal tab` (`0x09`)
- `\n` for the character `new line` (`0x0a`)
- `\0` for the character `null` (`0x00`)
- `\{DIGITS}`, where `DIGITS` is the ASCII decimal representation of any scalar value. `DIGITS` must consist of one to six characters.

When decoding, reading either a literal or an escape sequence that does not correspond to a Unicode scalar value is an *error*. In particular, Unicode code points that are not scalar values are not allowed, even when they form valid surrogate pairs.

A string whose content is valid UTF-8 can also be encoded without escape sequences: one to 256 `@`, followed by `"`, followed by valid UTF-8, followed by `"`, followed by the same number of `@` as at the beginning of the string encoding.

When decoding, reading a string consisting of more than `(2^63) - 1` bytes is an *error*.

#### Maps

A map is encoded as a comma-separated list of the contained pairs (see below), enclosed between braces `{` and `}`. An optional trailing comma before the closing brace is allowed. Any amount of whitespace can be placed between braces, contained values, and commas. The list may contain at most `(2^63) - 1` entries.

An entry is encoded as the encoding of the key, followed by any amount of whitespace, followed by a `:` followed by any amount of whitespace, followed by the encoding of the value.

When decoding multiple entries with identical keys, the later entry replaces the previous entry in the map.

#### Sets

A set (reminder: a map which maps all keys to `nil`) can also be encoded as a comma-separated list of the encodings of the keys, enclosed between `@{` and `}`. An optional trailing comma before the closing brace is allowed. Any amount of whitespace can be placed between braces, contained values, and commas. The list may contain at most `(2^63) - 1` entries.

When decoding, duplicate values are allowed and are simply discarded (the logical model does *not* allow multisets).

### Compact Encoding

The compact encoding uses techniques similar to [cbor](https://tools.ietf.org/html/rfc7049) to make it space-efficient and easy to parse. There is no concept of whitespace (and thus comments) in the compact encoding. The recommended file extension is `.cvv`.

Values are encoded as a single byte that tags the kind of value, sometimes followed by more, actual data. The first bit of all tags is a one, so that compact and human-readable codes can immediately be recognized. This is followed by three bits indicating the kind of the value. The remaining four bits carry additional information about the following data.

Some values have multiple valid encodings. Implementations are strongly encouraged to always use the shortest possible encoding, but reading an overlong code is *not* a decoding error.

#### Nil

`nil` is encoded as the tag `0b1_010_1100`, followed by no additional data.

#### Booleans

`false` is encoded as the tag `0b1_010_1101`, followed by no additional data.

`true` is encoded as the tag `0b1_010_1110`, followed by no additional data.

#### Floats

Floats are encoded as the tag `0b1_010_1111`, followed by the eight bytes of the float (sign, exponent, fraction in that order). NaN can be encoded as any valid IEEE 754 NaN (arbitrary sign, exponent all ones, remaining bytes arbitrary but non-zero).

#### Ints

Ints are encoded as the tag `0b1_011_xxxx`, where the least significant four bits and the following bytes are determined as follows:

- for least significant bits strictly less than `0b1100`, the bits themselves represent the encoded int (in the range from zero to eleven), no more bytes follow the tag
- for least significant bits `0b1100`, the tag is followed by a single byte, which encodes the int as two's complement (ranging from `-(2^7)` to `(2^7) - 1`)
- for least significant bits `0b1101`, the tag is followed by two bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^15)` to `(2^15) - 1`)
- for least significant bits `0b1110`, the tag is followed by four bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^31)` to `(2^31) - 1`)
- for least significant bits `0b1111`, the tag is followed by eight bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^63)` to `(2^63) - 1`)

#### Strings

Strings are encoded as the tag `0b1_100_xxxx`, where the least significant four bits and the following bytes are determined as follows:

- for least significant bits strictly less than `0b1100`, the bits themselves represent the length of the string, the tag is followed by that many bytes
- for least significant bits `0b1100`, the tag is followed by a single byte, which encodes the length of the string, followed by that many bytes of
- for least significant bits `0b1101`, the tag is followed by two bytes in big-endian order, which encode the length of the string, followed by that many bytes
- for least significant bits `0b1110`, the tag is followed by four bytes in big-endian order, which encode the length of the string, followed by that many bytes
- for least significant bits `0b1111`, the tag is followed by eight bytes in big-endian order, which encode the length of the string, followed by that many bytes

When decoding, reading a length of more than `2^63 - 1` is an *error*.

#### Arrays

Arrays are encoded just like strings, with the following differences:

- they use the tags `0b1_101_xxxx`
- the length is not given in bytes but in items
- after the length, the code is followed by the compact encodings of all items in order

#### Sets

Sets are encoded just like arrays, with the following differences:

- they use the tags `0b1_110_xxxx`
- when decoding, duplicate values are allowed but are simply discarded

#### Maps

Maps are encoded just like arrays, with the following differences:

- they use the tags `0b1_111_xxxx`
- the length is not given in items but in entries
- after the length, the code is followed by the encodings of all entries, where an entry is encoded as a compact encoding of its key followed by a compact encoding of its value
- when decoding multiple entries with identical keys, the later entry replaces the previous entry in the map

### Hybrid Encoding

The human-readable encoding and the compact encoding are completely disjunct, the hybrid encoding simply allows each (possibly nested) value to be encoded in either way. This encoding should be preferred for programs which read valuable values as input. The recommended file extension is `.hvv`.

### Canonic Encoding

A canonic encoding is one where there is a one-to-one correspondence between values and codes. The valuable value canonic encoding is a subset of the compact encoding, obtained through the following restrictions:

- ints, arrays and maps must use the shortest possible encodings for their length/size
- NaN must be encoded as the tag for a float followed by 64 1-bits
- strings are always encoded as regular arrays
- sets are always encoded as regular maps
- map entries must be sorted ascendingly by their keys, according to the linear order, each key occuring at most once
