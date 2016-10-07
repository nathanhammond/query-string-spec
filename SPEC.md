#### Canonical State

JSON, as a near-universal format in the HTTP world, is ipso facto the best in-memory representation of query strings as a data structure. It is easily and consistently parsed in every language, it supports the set of features needed to model a nested query string, and it also doesn't support additional features in a situation where every feature results in new edge cases.

#### General Requirements

Interoperability must be a primary concern of a query string library. Since this is a format native to the web we should take into account the limitations of the underlying system to gracefully represent content.

##### Spaces

The space `" "` character is specially handled inside of query strings per [HTML's form specification](http://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.1). This means that any space character must be converted to the `+` character. Any library should first escape any existing `+` characters and then convert all spaces to `+`.

##### Quotes

HTML uses quotes as a delimiter. Keys and values inside of the query string may contain `"` or `'` which make it more difficult to parse. There is _no_ special accounting for quotes inside of this specification. They will remain unescaped. This means that if you must have a quote present in either key or value in HTML you must transform it to either `&quot;` or `&#39;` for it to function equivalently to library code.

#### Edge Cases

Rather than explore an implementation of a nested parser–which is necessarily much larger and more complicated than its simplistic and collapsing counterparts–we'll enumerate the edge cases which must be addressed.

Each of these edge cases presents a possibly ambiguous scenario which the serializers and deserializers (across multiple libraries!) should be consistent. The transformation into a query string format is _necessarily_ lossy for most data; but that lossiness should be consistent and predictable.

##### Data Type Lossiness

By the time all of the information has been turned into a query string we lose all in-memory type information. From the JSON spec that means we lose the following values:

- `number`
- `true` and `false`
- `null`

All of these values are supported by JSON and should have a specification as to what their expected behavior is to allow libraries to produce useful values.

###### Numbers

Numbers should be turned into strings on serialization. On deserialization, fields of only numeric characters should remain strings and the consuming application should correctly handle type transformation.

- `{"num": 1234}` => `"num=1234"`
- `"num=1234"` => `{"num": "1234"}` 

###### Booleans

Booleans should be turned into strings on serialization. The string `"1"` maps to `true` and the string `"0"` maps to `false`. On deserialization these should remain strings and the consuming application should correctly handle type transformation.

- `{"truthy": true, "falsey": false}` => `"truthy=1&falsey=0"`
- `"truthy=1&falsey=0"` => `{"truthy": "1", "falsey": "0"}` 

###### Null

`null` values should be serialized to just having their key present in the output. Naked keys in the serialized form should be set back to `null`. If you wish to not have a key present in the output you must remove it from the object prior to serialization.

- `{"key": null}` => `"key"`
- `"key"` => `{"key": null}` 

##### Empty Strings

Empty strings turn out to be one of the trickier edge cases to handle correctly. Both keys and values may be set to an empty string.

- `{"key": ""}` => `"key="`
- `"key="` => `{"key": ""}`
- `{"": "value"}` => `"=value"`
- `"=value"` => `{"": "value"}` 

##### Multiple Key Occurrences

Considering that the canonical form for this mental model is a JSON object, this is an edge case that most libraries will not encounter as it can only come from manual or improper serialization. However, behavior to handle it should be consistent. The value of the _last_ occurrence of the key is what the key should be set to.

- `"a=1&a=2&a=3"` => `{"a": "3"}`

#### Nesting

All of these other patterns are on a key-by-key basis. There also need to be annotations inside of the key to make it possible to identify if it is part of a nested structure. The consensus notation for that is to use square brackets: `[` and `]`. They're unfortunately overloaded and used to designate both objects and arrays depending upon context.

##### Arrays

Arrays are denoted by square brackets adjacent to a previous key. The lack of a key denotes `push` ordering.

```javascript
var deserialized = {
  "colors": [
    "orange",
    "rebeccapurple"
  ]
}

var serialized = "colors[]=orange&colors[]=rebeccapurple"
```

##### Objects

Objects are serialized with their key names wrapped in square brackets. The entire key path to the value is thus specifed in the left hand side of the assignment. 

```javascript
var deserialized = {
  "colors": {
    "foreground": "orange",
    "background": "rebeccapurple"
  }
}

var serialized = "colors[foreground]=orange&colors[background]=rebeccapurple"
```

##### Edge Cases

The most obvious edge case is serializing a key which contains `[` or `]`. To support this we require that all square brackets be escaped 

- `{"[markdownlink]": "fragment"}` => `%5Bmarkdownlink%5D=fragment`

You _may not_ sort keys during the deserialization process as that removes information about key ordering. If the application desires sorted key ordering this must be done before corssing the boundary into the query string library as it is an application concern.

There are two pernicious side effects from the election to overload the square brackets to support both arrays and objects. First, arrays with only one value are indistinguishable from objects with an empty string `""` for a key. By rule we will always assume that the result is an array until we have evidence to the contrary. Once we have evidence to the contrary we collapse the array to the last element as per the multiple key occurrences rule. 

- `a[]=what` => `{"a": ["what"]}`
- `a[]=what&a[]=value` => `{"a": ["what", "value"]}`
- `a[]=what&a[subkey]=is&a[]=this` => `{"a": {"": "this", "subkey": "is"}}`

###### Nested Arrays

Nested arrays have different handling in different systems. Because of the two separate patterns in existence both must be supported on deserialization, and either strategy may be selected for serialization.

- The library must be able to intuit from numeric-only, zero-indexed keys that the result should be an array and not an object.
- Empty square brackets shall be treated as a `push` onto an array. Because we can guarantee ordering of arrays during serialization this is safe.

The tradeoff in supporting the numeric index approach is that it is impossible to represent an object with numeric-only keys starting at zero in a query string: `{"a": {"0": 0, "1": 1, "2": 2}}` and it requires additional logic during deserialization.

Since the object will simply be converted to an array the workaround in the consuming language will depend on its semantics. For example, in JavaScript this is a non-issue as it coerces numbers to strings when attempting to look them up in an object:

```javascript
var a = { "0": 0, "1": 1, "2": 2 }
a[1] === 1; // => true
a["1"] === 1; // => true
```

And JavaScript coerces strings to numbers when attempting to look them up in an array:

```javascript
var a = [0,1,2];
a[1] === 1; // => true
a["1"] === 1; // => true
```

Using only square brackets without indices has the tradeoff that a user may reorder keys in a way that results in an additional item being inserted into an array (by not having all array keys adjacent in the query string). If so, this manual manipulation is incorrect. Since manual manipulation is the only way this can go wrong the application should treat it the same as a validation error in which the user provides incorrect data. 

**PHP**

- **Serialize**: `{"a": ["one", [1,2,3], "three"]}` => `a[0]=one&a[1][0]=1&a[1][1]=2&a[1][2]=3&a[2]=three`
- **Deserialize (PHP)**: `a[0]=one&a[1][0]=1&a[1][1]=2&a[1][2]=3&a[2]=three` => `{"a": ["one", [1,2,3], "three"]}`
- **Deserialize (Rack)**: `a[]=one&a[][]=1&a[][]=2&a[][]=3&a[]=three` => `{"a": ["one", ["1"], ["2"], ["3"], "three"]}`

**Rack**

- **Serialize**: `{"a": ["one", [1,2,3], "three"]}` => `a[]=one&a[][]=1&a[][]=2&a[][]=3&a[]=three`
- **Deserialize (Rack)**: `a[]=one&a[][]=1&a[][]=2&a[][]=3&a[]=three` => `{"a": ["one", null, null, null, "three"]}`
- **Deserialize (PHP)**: `a[0]=one&a[1][0]=1&a[1][1]=2&a[1][2]=3&a[2]=three` => `{"a": {"0": "one","1": {"0": "1", "1": "2", "2": "3"}, "2": "three"}}`

###### Adjacent Nested Arrays

In the square bracket "push" notation it is impossible to identify when adjacent values in a nested array are also arrays. For example, in a best-case scenario:

- **Serialize**: `{"a": ["one", [1,2,3], [4,5,6]]}` => `a[]=one&a[][]=1&a[][]=2&a[][]=3&a[][]=4&a[][]=5&a[][]=6`
- **Deserialize**: `a[]=one&a[][]=1&a[][]=2&a[][]=3&a[][]=4&a[][]=5&a[][]=6` => `{"a": ["one", [1,2,3,4,5,6]}`

In this case the library must adopt the monotonically increasing zero-indexed numeric keys for each array at that nesting depth.

###### Sparse Arrays

Sparse arrays are unsupported as they are not able to be represented in JSON.

###### Nested Objects

For nested objects keys should be assigned and added to the key stack. If an object is a child of an array its key may be either numeric or empty square brackets and should work correctly.

**PHP**

- `{"a": ["one", {"two": 2}, "three"]}` => `a[0]=one&a[1][two]=2&a[2]=three`
- `a[0]=one&a[1][two]=2&a[2]=three` => `{"a": ["one", {"two": 2}, "three"]}`

**Rack**

- `{"a": ["one", {"two": 2}, "three"]}` => `a[]=one&a[][two]=2&a[]=three`
- `a[]=one&a[][two]=2&a[]=three` => `{"a": ["one", {"two": 2}, "three"]}`