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

---

Special thanks to Matthew Beale, Miguel Camba, and Trent Willis for reviewing drafts of this post.