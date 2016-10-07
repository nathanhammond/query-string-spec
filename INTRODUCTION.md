# Query Strings

[Every time you've ever performed a search on Google](https://www.google.com/?q=query+string) you've used a query string. If you've ever built a website with `<form method="get">` you've used query strings. They've been [codified to exist since the dawn of HTTP](https://www.ietf.org/rfc/rfc1738.txt). You've possibly even been asked to parse a query string as an interview question.

In spite of their prevalence you may also have been told that every single web framework implements them differently; that the variance in strategies meant that attempting a grand unified query string specification is foolish:

> There are simply too many different implementations and too many permutations to account for!

Instinctively we _want_ for this to be true. Edge cases are _hard_. Even the lack of any serious attempts over the past twenty years to attempt to specify query string behavior implies the difficulty of the task.

Simply being hard doesn't make the effort futile. Why is this area so steeped in convention rather than specification?

## What is a query string?

There are two ways to read [RFC 3986](https://www.ietf.org/rfc/rfc3986.txt) which defines a generic syntax for URIs and specifies how to identify a query string. The first way to approach it is to read it as specified in the [ABNF](https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_Form):

``` 
unreserved = ALPHA / DIGIT / "-" / "." / "_" / "~"
pct-encoded = "%" HEXDIG HEXDIG
sub-delims = "!" / "$" / "&" / "'" / "(" / ")" / "*" / "+" / "," / ";" / "="
pchar = unreserved / pct-encoded / sub-delims / ":" / "@"

query = *( pchar / "/" / "?" )
```

This definition _dramatically_ limits the characters allowed in a query string. This is less than ideal in a world in which ISO-8859-1 is unable to represent the way in which the majority of the world communicates. Given that most modern implementations of URL parsers fully support Unicode characters inside of URIs this definition is far too strict–it ends up unnecessarily encoding a large portion of characters in URIs.

The other approach–_presented in the same document_–is that a query string is nothing more than the portion of a URI which matches this regular expression: `/\?([^#]*)/`. This is presented as only applying to well-formed URIs matching the ABNF but in practice has meant that the defining feature of a query string is its boundary characters: `?` and `#`.

All modern implementations should use the boundary character interpretation unless they have strict compatibility requirements for integrating with legacy systems which do not support a broader character set.

And that is a summary of the extent to which query strings are officially specified. Nothing about ampersands, equal signs, or key/value pairs. It's the lack of further specification which makes the rules for how query string libraries handle serialization and deserialization between these boundary characters the most intriguing part.

## Serialization & Deserialization

Serialization is the process of taking the canonical state of a data structure and reducing it to something (often a string) which can be stored and reconstructed (deserialized) later. There are really only a few features that matter in serialization:

- The serialized format should be lossless. Serialization and deserialization should ideally be idempotent: `X === D(S(X))`.
- The processing for serialization and deserialization should be efficient. It should limit processing costs along time and memory axes.
- The serialized state should be as small as possible.
- The serialized format should possibly provide a reasonable human user interface.

If you check most of those boxes you've achieved a great serialization format. JSON is widely used as a serialization format because of how well it fits these attributes. XML is also used for serialization yet often mocked for its space inefficiency–though in its base format it has the ability to achieve higher information fidelity. Almost all serialization formats make tradeoffs to achieve different goals–which helps to explain why we have so many different formats.

<a href="https://xkcd.com/927/"><img src="https://imgs.xkcd.com/comics/standards.png" title="Fortunately, the charging one has been solved now that we've all standardized on mini-USB. Or is it micro-USB? Shit." alt="Standards"></a>

It's the application of serialization and deserialization to query strings with differing mental models which helps to explain the distinct approaches to query string handling.

## Mental Models

I posit that the fundamental distinction between the approaches in handling query strings can be dissolved into an analysis of the library's attitude toward which state should be considered the canonical state. If the library author treats the query string as provided in the URL as the canonical state they will provide a series of manipulation functions whose consequences are string modification. This can take the form of functions like `addParameter(key, value)` which simply appends to the existing string. In this approach there *may not be* a serialized format, just helper functions to make it easier to interact with the query string.

The second mental model is to treat the query string as the serialized state and have a (much more complex) in-memory object as the canonical state. This mental model and approach is common in more-dynamic languages such as Ruby and JavaScript. It also maps remarkably well onto JSON as an in-memory format which is commonly used in these languages.

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

When serializing from a keyed data structure it is impossible to achieve an output that matches the above input because of key collisions. It must instead be serialized from an array of key/value tuples. This is a data structure which is easy to achieve in any language.

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

These are clearly operations which fully reverse each other in a lossless manner. Order remains consistent, keys and values remain consistent. This achieves the ideal for all serialization and deserialization algorithms but it does not, however, make it easy to work with the query string. The application developer will have to consistently scan the array of tuples to identify what the application should do.

### Collapsing

In a slight modification to the simplistic approach, there is a modified deserialization pattern which moves all of the items of the same key into a single key space. This collapsing allows for the in-memory representation to be a keyed data structure and makes it a bit easier to work with the data as an application developer needn't iterate over all key/value tuples prior to use inside of an application.

#### Deserialization

There is still no introspection into the keys themselves, it is done blindly. Collapsing occurs if the keys match and the values are stored in arrays at that particular key.

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

    // MODIFIED FROM SIMPLISTIC:
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
queryString === serialize(deserialize(queryString)); // false
```

This is because value ordering ends up being hoisted to the _first_ occurrence of the key in the query string. If key/value pair order is important you _must_ use the simplistic strategy. If ordering of values within a single key is the only concern or not a concern then you may choose to use the collapsing strategy. This also works well for "read-only" patterns where you don't need to modify and then re-serialize the query string.

### Nested

More-dynamic and HTTP-native languages have taken this one step further with the goal of storing structured data inside of the the query string. This Nested strategy should be considered the primary strategy and the simplistic and collapsing strategies should be considered deprecated by this pattern.

One could simply avoid all of this effort and simply storing JSON or any other serialized format inside of the query string, but that suffers from a poor user experience that doesn't leverage the largely lay-understandable (through sheer repetition) "common" query string format.

The best approach to serializing structured data into a query string is to provide annotations within only the key. The nested approach of course must be backed by an in-memory data structure as the canonical state.

### Less Common Strategies

There are a few unique implementations that exist clustered in the space of annotations within the value assigned to a key, sometimes in pair with special notation in the key as well to trigger even more complex serialization and deserialization strategies. This is categorically wrong. Storing your data's structure between key and value violates expectations. Further, customization of approaches has significant drawbacks in terms of interoperability and limiting representation of values.

[One such example of a custom strategy is the serialization of arrays into JSON strings in Ember.](https://github.com/emberjs/ember.js/blob/master/packages/ember-routing/lib/system/route.js#L431-L433) Demonstrating exactly how problematic these custom patterns are, [Ember's serialization is in direct conflict](https://github.com/emberjs/ember.js/issues/14174) with the approach taken by Ember's underlying routing layer provided by [`route-recognizer`](https://github.com/tildeio/route-recognizer), which is in conflict with the included [jQuery implementation](http://api.jquery.com/jquery.param/)–all three approaches shipped with every current Ember applications.

These one-off patterns result in much-greater system complexity and should be avoided. As part of this effort I intend to help the Ember community migrate to the nested strategy as specified above.

## Next Steps

The solution to this problem and the way out of the woods is of course standardizing behavior. However, there is no way to coerce existing implementations into supporting a specification and doing so would be in bad form since it would break existing applications.

I've begun the process of [writing a specification and test harness](https://github.com/nathanhammond/query-string-spec) to which you can contribute new test cases and identify behavior in the edge cases of individual implementations.

This work must be future-focused and will hopefully result in specification-compliant libraries in every language and framework. Claiming compatibility guarantees that your library will gracefully integrate with all other libraries which also pass the test suite.

---

Special thanks to Matthew Beale, Miguel Camba, and Trent Willis for reviewing drafts of this post.