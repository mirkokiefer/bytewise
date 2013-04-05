bytewise
========

This library defines a total order of possible data structures allowed in a keyspace and a binary encoding which sorts
bytewise in this order. The ordering is a superset of both the sorting algorithm defined by
[IndexedDB](http://www.w3.org/TR/IndexedDB/#key-construct) and the one defined by
[CouchDB](http://wiki.apache.org/couchdb/View_collation). This serialization makes it easy to take advantage of the
benefits of structural indexing on systems with fast but naïve binary indexing.


## Order of Supported Structures

This is the top level order of the various structures that may be encoded:

* `undefined`
* `null`
* `false`
* `true`
* `Number` (numeric)
* `Date` (numeric, epoch offset)
* `Buffer` (bitwise)
* `String` (lexicographic)
* `Set` (componentwise with elements sorted)
* `Array` (componentwise)
* `Map` (componentwise key/value pairs)
* `Function` (stringified lexicographic)


These specific structures can be used to serialize the vast majority of javascript values in a way that can be sorted in
an efficient, complete and sensible manner. Each value is prefixed with a type tag, and we do some bit munging to encode
our values in such a way as to carefully preserve the desired sort behavior, even in the precense of structural nested.

For example, negative numbers are stored as a different *type* from positive numbers, with its sign bit stripped and its
bytes inverted to ensure numbers with a larger magnitude come first. `Infinity` and `-Infinity` can also be encoded --
they are *nullary* types, encoded using just their type tag., same with `null` and `undefined`, and the boolean values
`false`, `true`, . `Date` instances are stored just like `Number` instances, but as in IndexedDB, `Date` sorts after
`Number` (including `Infinity`). `Buffer` data can be stored in the raw, and is sorted before `String` data. Then come
the collection types -- `Array`, `Object`, along with the additional types defined by es6: `Map` and `Set`. We can even
serialize `Function` values, reviving them in an isolated [Secure ECMAScript](https://code.google.com/p/es-
lab/wiki/SecureEcmaScript) context where they can't do anything but calculate.


## Unsupported Structures

This serialization accomodates a wide range of javascript structures, but it is not exhaustive. Objects or arrays with
reference cycles, for instance, cannot be serialized. `NaN` is also illegal anywhere in a serialized value, as its
presense is very likely indicative of an error. Moreover, sorting for `NaN` is completely nonsensical. (Similarly we may
want to reject objects which are instances of `Error`.) Invalid `Date` objects are also illegal. Attempts to serialize
any values which include these structures will throw a `TypeError`.


## Usage

`encode` serializes any supported type and returns a buffer, or throws if an unsupported structure is passed:
  
  ``` js
  var bytewise = require('bytewise');
  var assert = require('assert');
  function hexEncode(buffer) { return bytewise.encode(buffer).toString('hex') }

  // Many types can be respresented using only their type tag, a single byte
  // WARNING type tags are subject to change for the time being!
  assert.equal(bytewise.encode(undefined).toString('binary'), '\x10');
  assert.equal(bytewise.encode(null).toString('binary'), '\x11');
  assert.equal(bytewise.encode(false).toString('binary'), '\x20');
  assert.equal(bytewise.encode(true).toString('binary'), '\x21');

  // Numbers are stored in 9 bytes -- 1 byte for the type tag and an 8 byte float
  assert.equal(hexEncode(12345), '4240c81c8000000000');
  // Negative numbers are stored as positive numbers, but with a lower type tag and their bits inverted
  assert.equal(hexEncode(-12345), '41bf37e37fffffffff');

  // All numbers, integer or floating point, are stored as IEEE 754 doubles
  assert.equal(hexEncode(1.2345), '423ff3c083126e978d');
  assert.equal(hexEncode(-1.2345), '41c00c3f7ced916872');

  // Serialization preserves the sign bit by default, so 0 is distinct from (but directly adjecent to) -0
  assert.equal(hexEncode(-0), '41ffffffffffffffff');
  assert.equal(hexEncode(0), '420000000000000000');

  // We can even serialize Infinity and -Infinity, though we just use their type tag
  assert.equal(hexEncode(-Infinity), '40');
  assert.equal(hexEncode(Infinity), '43');

  // Dates are stored just like numbers, but with different (and higher) type tags
  assert.equal(hexEncode(new Date(-12345)), '51bf37e37fffffffff');
  assert.equal(hexEncode(new Date(12345)), '5240c81c8000000000');

  // Strings are as utf8 prefixed with their type tag
  assert.equal(hexEncode('foo'), '70666f6f');

  // That same string encoded in the raw
  assert.equal(bytewise.encode('foo').toString('binary'), '\x70foo')

  // Buffers are completely left alone, other than being prefixed with their type tag
  assert.equal(hexEncode(new Buffer('ff00fe01', 'hex')), '60ff00fe01');

  // Arrays are just a series of values terminated with a null byte
  assert.equal(hexEncode([ true, -1.2345 ]), 'a02141c00c3f7ced91687200');

  // When embedded in complex structures (like arrays) Strings and Buffers have their bytes shifted
  // to make way for a null termination byte to signal their end
  assert.equal(hexEncode([ 'foo' ]), 'a0706770700000');

  // That same string encoded in the raw -- note the 'gpp', the escaped version of 'foo'
  assert.equal(bytewise.encode(['foo']).toString('binary'), '\xa0\x70gpp\x00\x00');

  // The 0xff byte is used as an escape to encode 0xfe and 0xff bytes, preserving the correct collation
  assert.equal(hexEncode([ new Buffer('ff00fe01', 'hex') ]), 'a060ffff01fffe020000');

  // Complex types like arrays can be arbitrarily nested, and fixed-sized types never need a terminating byte
  assert.equal(hexEncode([ [ true, 'foo' ], -1.2345 ]), 'a0a02170677070000041c00c3f7ced91687200');

  // Objects are just string-keyed maps, stored like arrays: [ k1, v1, k2, v2, ... ]
  assert.equal(hexEncode({ foo: true, bar: -1.2345 }), 'b0706770700021706362730041c00c3f7ced91687200');
  ```


`decode` parses a buffer and returns the structured data, or throws if malformed:
  
  ``` js
  var samples = [
    'foo √',
    null,
    '',
    new Date('2000-01-01T00:00:00Z'),
    42,
    undefined,
    [ undefined ],
    -1.1,
    {},
    [],
    true,
    { bar: 1 },
    [ { bar: 1 }, { bar: [ 'baz' ] } ],
    -Infinity,
    false
  ];
  var result = samples.map(bytewise.encode).map(bytewise.decode);
  assert.deepEqual(samples, result);
  ```


`compare` is just a convenience bytewise comparison function:

  ``` js
  var sorted = [
    undefined,
    null,
    false,
    true,
    -Infinity,
    -1.1,
    42,
    new Date('2000-01-01T00:00:00Z'),
    '',
    'foo √',
    [],
    [ undefined ],
    [ { bar: 1 }, { bar: [ 'baz' ] } ],
    {},
    { bar: 1 }
  ];

  var result = samples.map(bytewise.encode).sort(bytewise.compare).map(bytewise.decode);
  assert.deepEqual(sorted, result);
  ```


## Use Cases


### Numeric indexing

This isn't surprisingly difficult to with vanilla LevelDB -- basic approaches require ugly hacks like left-padding
numbers to make them sort lexicographically (and is prone to overflow problems). This serializes solves this problem
correctly, taking advantage of properties of the byte sequences defined by the IEE 754 floating point standard.

### Namespaces and partitions

This is another really basic ammenity that isn't as easy out of the box as it should be.

We reserve the hightest byte as an abstract tag representing a high-key sentinal. When combined with structures like
arrays this allows you to faithfully request all values in portion of the beginning of an array without any leaky hacks.

### Document storage

It may be reasonably fast to encode and decode, but `JSON.stringify` is totally useless for storing objects as document
records in a way that is of any use for range queries, where LevelDB and its ilk excel. Our serialization allows you to
build indexes on top of your documents. Being unable to indexing purposes since it doesn't sort correctly. We fix that,
and even expand on the range of serializable types available.

### Multilevel language-sensitive comparison

You have a bunch of strings in a paritcular language-specific strings you want to index, but you are know sure *how*
sorted you need them -- queries may or may not care about case or punctionation differences, for instance. You can index
your string as an array of weights, most to least specific, and prefixed by collation language (since our values are
language-sensitive). There are [mechanisms available](http://www.unicode.org/reports/tr10/#Run-length_Compression) to
compress this array to keep its size reasonable.

### CouchDB-style "joins"

Build a view that colocates related subrecords, taking advantage of component-wise sorting of arrays to interleave them.
This is a technique I first saw [employed by CouchDB](http://www.cmlenz.net/archives/2007/10/couchdb-joins). More
recently [Akiban](http://www.akiban.com/) has formalized this concept of [table grouping](http://blog.akiban.com/how-
does-table-grouping-compare-to-sql-server-indexed-views/) and brought it the SQL world. Our collation extends naturally
to their idea of [hierarchical keys](http://blog.akiban.com/introducing-hkey/).

### Emulating other systems

Clients that wish to employ a subset of the full range of possible types above can preprocess values to coerce them into
the desired simpler forms before serializing. For instance, if you were to build CouchDB-style indexing you could round-
trip values through a `JSON` encode cycle (to get just the subset of types supported by CouchDB) before passing to
`encode`, resulting in a collation that is identical to CouchDB. Emulating IndexedDB's collation would require
preprocessing away `Buffer` data and `undefined` values. (TODO what else? Does it normalize `-0` values to `0`?)

### Embedding in the browser

While this particular serialization is only useful for binary indexing, the collation defined can easily be extended to
embed inside indexeddb. At the top level all values would be arrays with the type tag as the first value. Any binary
data would have be transcoded to a string in a manner that preserves sort order. Otherwise we can lean on IndexedDB's
default sort behavior as much as possible.


## Future

### Generic collections

The ordering chosen for some of the types is somewhat arbitrary. It is intentionally structured to support those sorts
defined by CouchDB and IndexedDB but there might be more logical placements, specifically for BUFFER, SET, and FUNCTION,
which aren't defined in either. It may be beneficial to fully characterize the distinctions between collections that
affect collation.
  
One possible breakdown for collection types:

* sorted set (order unimportant and thus sorted using standard collation)
  * sorted multiset, duplicates allowed
* ordered set (order-preserving with distinct values)
  * ordered multiset, duplicates allowed (an array or tuple)
* sorted map (keys as sorted set, objects are string-keyed maps)
  * sorted multimap (keys as sorted multiset), duplicates allowed
* ordered map (keys as ordered set)
  * ordered multimap (keys as ordered multiset), duplicates allowed


The primary distinction between collections are whether their items are unary (sets or arrays of elements) or binary
(maps of keys and values). The secondary distinction is whether the collection preserves the order of its elements or
not. For instance, arrays preserve the order of their elements while sets do not. Maps typically don't either, nor do
javascript objects (even if they appear to at first). These are the two bits which characterize collection types that
globally effect the sorting of the types.

There is a third characterizing bit: whether or not duplicates are allowed. The effect this has on sort is very
localized, only for breaking ties between two otherwise identical keys -- otherwise records are completely interwoven
when sorted, regardless of whether duplicates are allowed or not.

We may want unique symbols to signal these characterizing bits for serialization.

We probably want hooks for custom revivers.

Sparse arrays could be modeled with sorted maps of integer keys, but should we use a trailer to preserve this data?

This is very close to a generalized total order for all possible data structure models.

### Performance

Encoding and decoding is surely slower than the native `JSON` functions, but there is plenty of room to narrow the gap.
Once the serialization stabilizes a C port should be straitforward to narrow the gap further.

### Streams

Where this serialization should really shine is streaming. Building a from stream from individually encoded elements
should require little more than strait concatenation, and parsing a stream would be the same as parsing an array.

It's not typically useful for each and every array (and object) to be emitted as a stream, but it's not always the case
that a stream purely consists of elements in a top-level array. There can be other "pivot points" where streams make
sense. For instance, an array of records might be sensible to stream... but what if the record has a field that could be
quite large (say, the binary contents of an image)? It should be possible to add a handler for this field in the parser,
and when it reaches this kind of path it will emit a stream. The handler could have water-mark hints for how to break up
the elements. If the value targeted to stream is a `String` or `Buffer` we'd have to be careful with unescaping, and for
`Strings` specifically we'd also have to ensure we don't hack up multibyte characters.

Alternatively we could define another bit for collections -- the *streaming* bit -- that lets the creator dictate what
values are intended to stream. Of course, in order to avoid buffering all streams the consumer would still have to
handle all streams during parsing. Still, this does seem like a real characterization flag for collection types, and
like the *unique* flag, it shouldn't affect the global type sort so it should appear in the trailer bit. This would mean
we'd need to steal another few bits in the flat sequence escapement, which isn't much of a problem. Worse, it would mean
in order to tell when the stream bit is set the whole stream would have needed to be buffered, but this may not be such
a bad thing, considering we'd need a pre-defined handler in order to avoid buffering anyway.

With a sensible path language we could combine these two approaches. During parsing consumers could add a general
handler is given the path and a stream instance every time a stream collection is hit. A consumer could also handlers
specific to particular paths, the handler only being called when the path matches, and the underlying value is coerced
into a stream, regardless of what type it is (though it may not make sense to bother building streams for scalar types).
This could be a special case of a more general path handler that doesn't do stream coercion. If handlers are added no
final parsed value may be needed, in which case no callback should be passed to the parser. Or all of these approaches
could be combined -- stream handlers, path handlers, and a fully parsed value passed to the callback when complete.


## License

[MIT](http://deanlandolt.mit-license.org/)
