# Query Strings

[Every time you've ever performed a search on Google](https://www.google.com/?q=query+string) you've used a query string. If you've ever built a website with `<form method="get">` you've used query strings. They've been [codified to exist since the dawn of HTTP](https://www.ietf.org/rfc/rfc1738.txt). You've possibly even been asked to parse a query string as an interview question.

In spite of their prevalence you may also have been told that every single web framework implements them differently; that the variance in strategies meant that attempting a grand unified query string specification is foolish:

> There are simply too many different implementations and too many permutations to account for!

Instinctively we _want_ for this to be true. Edge cases are _hard_. Even the lack of any serious attempts over the past twenty years to attempt to specify query string behavior implies the difficulty of the task.

But simply being hard doesn't make the effort futile. Why is this area so steeped in convention rather than specification?

## What is a query string?

There are two ways to read [RFC 3986](https://www.ietf.org/rfc/rfc3986.txt) which defines a generic syntax for URIs to figure out what a query string is. The first way to approach it is to read it as specified in the [ABNF](https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_Form):

``` 
unreserved = ALPHA / DIGIT / "-" / "." / "_" / "~"
pct-encoded = "%" HEXDIG HEXDIG
sub-delims = "!" / "$" / "&" / "'" / "(" / ")" / "*" / "+" / "," / ";" / "="
pchar = unreserved / pct-encoded / sub-delims / ":" / "@"

query = *( pchar / "/" / "?" )
```

This definition *dramatically* limits the values allowed in a query string. This is also far less than ideal–it ends up encoding a large portion of characters in URIs unnecessarily. Most modern implementations fully support Unicode characters inside of URIs–making encoding characters almost entirely superfluous.

The other approach presented in the same document is that a query string is nothing more than the portion of a URL which matches this regular expression: `/\?([^#]*)/`. This is presented as only applying to well-formed URIs matching the ABNF, but in practice it has meant that the defining feature of a query string is its boundary characters: `?` and `#`.

All modern implementations should use the boundary character interpretation unless they have strict compatibility requirements for integrating with legacy systems which do not support the broader character set.

How query string libraries handle what happens between these boundary characters is actually the most intriguing part. But, before we analyze the approaches, we should take a moment to detour through the topic of serialization and deserialization.

## Serialization & Deserialization

Serialization is the concept of taking the canonical state and reducing it to something that can be stored and reconstructed (deserialized) later. There are really only a few features that matter in serialization:

- The serialized format should be lossless. Serializing an object to a string and then deserializing that string back into an object should ideally be idempotent.
- The serialization and deserialization process should be efficient (time and space).
- The serialized format should be as small as possible.
- The serialized format may need to provide a reasonable user interface.

If you check most of those boxes you've achieved a great serialization format. JSON is widely used because of how well it fits these attributes. XML is mocked for its space inefficiency–though in its base format it has the ability to achieve higher information fidelity. Almost all serialization formats make tradeoffs to achieve different goals which helps to explain why we have so many different formats.

<a href="https://xkcd.com/927/"><img src="https://imgs.xkcd.com/comics/standards.png" title="Fortunately, the charging one has been solved now that we've all standardized on mini-USB. Or is it micro-USB? Shit." alt="Standards"></a>

But it's the application of serialization and deserialization with differing mental models which helps to explain the three distinct approaches to query string handling. There are three primary serialization and deserialization strategies which have emerged but none of those  and any query string library should support all three.

## Mental Models

I posit that the fundamental distinction between the approaches in handling query strings can be dissolved into an analysis of the library's attitude toward what state should be considered the canonical state. If the library author treats the query string as provided in the URL as the canonical state they will provide a series of manipulation functions whose consequences are string modification. There *may not be* a serialized format in this approach, just helper functions to make it easier to interact with the query string.

The second mental model is to treat the query string as the serialized state and have a (much more complex) in-memory object as the canonical state. This mental model and approach is common in more-dynamic languages such as Ruby and JavaScript. It also maps remarkably well onto JSON as an in-memory format which is commonly found in these languages.

That there are three common implementation strategies and only two separate mental models is for reasons of pragmatism in implementation of the string manipulation tools when the query string is treated as the canonical state.

## Implementation Strategies

After reviewing more than 20 different implementations across multiple languages and frameworks I've distilled the strategies into four clusters. One of these clusters is wrong, three of these clusters are reasonable, but two should be considered deprecated if supported in any query string library.

Each of these strategies will have edge cases based upon implementation details but that doesn't disallow best effort. In fact, differences should be considered interoperability bugs and be addressed by each of the implementations.

### Simplistic

The mental model for the simplistic approach is that the query string itself is actually the canonical form. Each key and each value is treated as a block into which the parser does not introspect.

#### Deserialization

Deserialization for this approach returns an array of key/value tuples no matter what the content of the query string.

``` javascript
var serialized = "yo=dawg&yo=yodawg&yo=&=1&present&maybe%5B%5D=1&maybe%5B%5D=2";

function deserialize(serialized) {
  return serialized.split('&').map(function(stringpair) {
    var tuple = stringpair.split('=').map(decodeURIComponent);
    if (tuple.length === 1) {
      // NOTE: implementation specific. Sometimes:
      tuple.push(null);
    }
    return tuple;
  });
}

var deserialized = deserialize(serialized);

deserialized === [
  ["yo", "dawg"],
  ["yo", "yodawg"],
  ["yo", ""],
  ["", "1"],
  ["present", null],
  ["maybe[]", "1"],
  ["maybe[]", "2"]
];
```

Multiple occurrences of the same key do nothing special. Notation inside of the keys or values does nothing.

#### Serialization

When serializing from a keyed data structure it is impossible to achieve this output. It must be serialized from an array of key/value tuples. This is a data structure which is easy to achieve in any language.

``` javascript
var deserialized = [
  ["yo", "dawg"],
  ["yo", "yodawg"],
  ["yo", ""],
  ["", "1"],
  ["present", null],
  ["maybe[]", "1"],
  ["maybe[]", "2"]
];

function serialize(deserialized) {
  return deserialized.reduce(function(previous, current) {
    if (previous != '') {
      previous += '&';
    }
    if (current[1] === null) {
      current.length = 1;
    }
    current = current.map(encodeURIComponent);
    previous += current.join('=');
    return previous;
  }, '');
}

var serialized = serialize(deserialized);

serialized === "yo=dawg&yo=yodawg&yo=&=1&present&maybe%5B%5D=1&maybe%5B%5D=2";
```

These are clearly operations which fully reverse each other in a lossless manner. Order remains consistent, keys and values remain consistent. This achieves the ideal for all serialization and deserialization algorithms but it does not, however, make it easy to work with the query string.

### Collapsing

In a slight modification to the simplistic approach, there is a modified deserialization pattern which moves all of the items of the same key into a single key space. This collapsing allows for the in-memory representation to be a keyed data structure and makes it a bit easier to work with the data as you needn't iterate over all key/value tuples prior to processing.

#### Deserialization

There is still no introspection into the keys themselves, but the values are collapsed into arrays.

``` javascript
var serialized = "yo=dawg&yo=yodawg&yo=&=1&present&maybe%5B%5D=1&maybe%5B%5D=2";

function deserialize(serialized) {
  var result = {};

  // Rely on external variable to prevent unnecessary reduce.
  serialized.split('&').map(function(stringpair) {
    var tuple = stringpair.split('=').map(decodeURIComponent);
    if (tuple.length === 1) {
      // NOTE: implementation specific. Sometimes:
      tuple.push(null);
    }

    // MODIFIED FROM ABOVE:
    var key = tuple[0];
    var value = tuple[1];

    if (result[key]) {
      if (Array.isArray(result[key])) {
        result[key].push(value);
      } else {
        // Convert the existing value into an array of values.
        result[key] = [result[key], value];
      }
    } else {
      // NOTE: implementation specific. Sometimes:
      result[key] = value;
    }
  });

  return result;
}

var deserialized = deserialize(serialized);

deserialized === {
  "yo": [
    "dawg",
    "yodawg",
    ""
  ],
  "": "1",
  "present": null,
  "maybe[]": [
    "1",
    "2"
  ]
};
```

This at first glance appears to introduce irreconcilable differences in the fact that the in-memory data structure may have an array for every key, or only for keys which have multiple values.

#### Serialization

However, it's simple to implement a serialization function which works for any possible deserialization strategy regarding arrays in the previous code example. This makes the in-memory data structure's layout moot as the interchange format between libraries using the collapsing strategy is consistent.

``` javascript
var deserialized = {
  "yo": [
    "dawg",
    "yodawg",
    ""
  ],
  "": "1",
  "present": null,
  "maybe[]": [
    "1",
    "2"
  ]
};

function serialize(deserialized) {
  var keys = Object.keys(deserialized);

  return keys.reduce(function(previous, key) {
    if (previous != '') {
      previous += '&';
    }
    var values = deserialized[key];
    key = encodeURIComponent(key);

    if (values === null) {
      return previous + key;
    }

    if (Array.isArray(values)) {
      values = values.map(function(value) {
        if (value === null) {
          return key;
        } else {
          return key + '=' + encodeURIComponent(value);
        }
      });
      return previous + values.join('&');
    }

    // Else:
    return previous + key + '=' + encodeURIComponent(values);
  }, '');
}

var serialized = serialize(deserialized);

serialized === "yo=dawg&yo=yodawg&yo=&=1&present&maybe%5B%5D=1&maybe%5B%5D=2";
```

Unfortunately, this pair of serializer and deserializer are not guaranteed to losslessly process a query string. In other words,

``` javascript
var queryString = "yo=dawg&yo=yodawg&present";
queryString === serialize(deserialize(queryString)); // true

queryString = "yo=dawg&present&yo=yodawg";
queryString !== serialize(deserialize(queryString)); // true
```

This is because value ordering ends up being hoisted to the _first_ occurrence of the key in the query string. If key/value pair order is important you _must_ use the simplistic strategy. If ordering of values within a single key is the only concern or not a concern then you may choose to use the collapsing strategy.

### Nested

More-dynamic and HTTP-native languages have taken this one step further with the goal of storing structured data inside of the the query string. This Nested strategy should be considered the primary strategy and the simplistic and collapsing strategies should be considered deprecated by this pattern.

One could simply avoid all of this effort and simply storing JSON or any other serialized format inside of the query string, but that suffers from a poor user experience that doesn't leverage the largely lay-understandable (through sheer repetition) "common" query string format.

The best approach to serializing structured data into a query string is to provide annotations within only the key. The nested approach of course must be backed by an in-memory data structure as the canonical state.

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

---

### Less Common Strategies

There are a few unique implementations that exist clustered in the space of annotations within the value assigned to a key, sometimes in pair with special notation in the key as well to trigger even more complex serialization and deserialization strategies. This is categorically wrong. Storing your data's structure between key and value violates expectations. Further, customization of approaches has significant drawbacks in terms of interoperability and limiting representation of values.

[One such example of a custom strategy is the serialization of arrays into JSON strings in Ember.](https://github.com/emberjs/ember.js/blob/master/packages/ember-routing/lib/system/route.js#L431-L433) Demonstrating exactly how problematic these custom patterns are, [Ember's serialization is in direct conflict](https://github.com/emberjs/ember.js/issues/14174) with the approach taken by Ember's underlying routing layer provided by [`route-recognizer`](https://github.com/tildeio/route-recognizer), which is in conflict with the included [jQuery implementation](http://api.jquery.com/jquery.param/)–all three approaches shipped with every current Ember applications.

These one-off patterns result in much-greater system complexity and should be avoided. As part of this effort I intend to help the Ember community migrate to the nested strategy as specified above.

## Next Steps

The solution to this problem and the way out of the woods is of course standardizing behavior. However, there is no way to coerce existing implementations into supporting a specification and doing so would be in bad form since it would break existing applications.

I've begun the process of [writing a specification and test harness](https://github.com/nathanhammond/query-string-spec) to which you can contribute new test cases and identify behavior in the edge cases of individual implementations.

This work must be future-focused and will hopefully result in specification-compliant libraries in every language and framework. Claiming compatibility guarantees that your library will gracefully integrate with all other libraries which also pass the test suite.

### Ecosystem Review

To ensure that we actually specify common behavior and aren't building "yet another" query string serializer and deserializer it's necessary to survey the state of the ecosystem to make sure that we track relatively close to existing de facto behavior. I realized after data collection that I hadn't accounted for character set support, so that is left to fill in later.

#### Java

##### [`org.apache.http.client.utils.URLEncodedUtils`](https://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/client/utils/URLEncodedUtils.html)

- Pattern: Simplistic
- Character Set: ? 

##### [`android.net.Uri`](https://developer.android.com/reference/android/net/Uri.html)

- Pattern: Collapsing
- Character Set: ? 

##### [`javax.servlet.ServletRequest`](http://docs.oracle.com/javaee/1.3/api/javax/servlet/ServletRequest.html)

- Pattern: Collapsing
- Character Set: ? 

##### [`org.springframework.web.util.UriComponentsBuilder`](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/util/UriComponentsBuilder.html)

- Pattern: Collapsing
- Character Set: ? 

#### JavaScript

##### [qs npm module](https://www.npmjs.com/package/qs)

Most specifically used by [`Express`](https://expressjs.com/).

- Pattern: Nested
- Character Set: ? 

##### [Node.js Standard Library](https://nodejs.org/api/querystring.html)

- Pattern: Collapsing
- Character Set: ? 

##### [query-string npm module](https://github.com/sindresorhus/query-string)

- Pattern: Collapsing
- Character Set: ? 

##### [`jQuery.param`](http://api.jquery.com/jquery.param/)

This is, by rule, the client-side standard with [approximately 70% usage across the top 100,000 websites](http://trends.builtwith.com/javascript/jQuery). *It does not implement a deserializer.*

- Pattern: Nested
- Character Set: ? 

##### Ember.js

- Pattern: Custom value introspection.
- Character Set: ? 

##### route-recognizer

- Pattern: Single-level nested.
- Character Set: ? 

#### Ruby

##### [Rack](http://rack.github.io/)

- Pattern: Nested
- Character Set: ?
- [Implementation](https://github.com/rack/rack/blob/master/lib/rack/query_parser.rb) 

#### PHP

##### Standard Library

Includes [`parse_str`](http://php.net/manual/en/function.parse-str.php) and [`http_build_query`](http://php.net/manual/en/function.http-build-query.php) in the standard library.

- Pattern: Nested
- Character Set: ? 

#### Python

##### Standard Library

Includes [`urlparse.parse_qs`](https://docs.python.org/2/library/urlparse.html#urlparse.parse_qs) and [`urlparse.parse_qsl`](https://docs.python.org/2/library/urlparse.html#urlparse.parse_qsl).

- Pattern: Collapsing
- Character Set: ? 

##### [`querystring_parser`](https://github.com/bernii/querystring-parser).

- Pattern: Nested
- Character Set: ?
