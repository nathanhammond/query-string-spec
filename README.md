# Query String Specification

To this point there has been no standardization of how a query string library is supposed to work. This test suite exists to make it possible to deliver a query string library which is fully interoperable with any other implementation which also supports this specification.

To try and move every existing system to this specification seems, generously, hubristic. However, the number of systems interacting with each other over HTTP has ballooned to the point where interoperability has become paramount. Specifying the behavior of those systems in their corner cases becomes a task of tremendous importance to prevent virtually undetectable bugs from appearing within the complex systems which build on top of a "simple" query string parser.

To identify what this specification should be lots of research was done as to the existing behavior in the ecosystem. This is a specification by consensus, not by fiat.

## Reporting Issues

Reporting issues with the specification can be done via GitHub issues. _All issues **must** include a test case and example results from multiple implementations._

## Testing

_Please do not open issues against your favorite libraries if they do not happen to pass this test suite._

There are two ways to run the specification tests. The first is to use the built-in harness inside of this repository which were used during the initial ecosystem survey. The second is to write your own harness and import the tests themselves into your own test suite.

### Writing an Adapter

Your adapter must be invocable with this pattern:

```
$ ./test [method] [strategy] [teststring]
```

- `method` must be one of `stringify` or `parse`.
- `strategy` must be one of `simplistic`, `collapsing`, or `nested`.
- `teststring` will be:
  - a string when invoking `parse`.
  - a serialized JSON object when invoking `stringify`.
- It must output the result to `stdout` followed by a new line.
- It must exit with `0` if it runs to completion without error, and a positive integer if an error occurs.

Examples can be found in the `scripts` folder. Pull requests for additional adapters are accepted.

### Importing the Tests

The tests are listed as a six-element tuple and themselves serialized into JSON.

1. `id`:
 - a string identifer.
2. `method`:
  - one of `stringify` or `parse`.
2. `strategy`:
  - one of `simplistic`, `collapsing`, or `nested`.
3. `teststring`: 
  - a string when `method` is `parse`.
  - a serialized JSON object when `method` is `stringify`.
4. `output`:
  - the expected output from the method under test.
5. `description`
  - a description of the test.

The logic for implementation can be reviewed inside of `tests/run`.

## Who is behind this effort?

This effort is being spearheaded by [Nathan Hammond](https://twitter.com/nathanhammond) as part of the Ember community's effort to fully standardize and specify their routing behavior. The reason for this approach inside of the Ember community is neatly summed up by [Edward Faulkner](https://twitter.com/eaf4):

> ...this is a problem complex enough to justify having one well-thought-out community solution. It's not the easy stuff that needs to be standardized, it's the hard stuff. The whole point of Ember is to identify that kind of problem and solve it together. Even though general solutions are harder to find than one-off "it works for me" solutions, it is massively more valuable in the long run. -[Edward Faulkner](https://github.com/emberjs/rfcs/pull/143#issuecomment-244612714)

We look forward to the day when users of HTTP are able to blithely ignore the minutae of URL processing and how that impacts the applications that they build.