Autodoc
=======

[![Build Status](https://travis-ci.org/dtao/autodoc.png)](https://travis-ci.org/dtao/autodoc)

Autodoc helps you keep your JavaScript project nice and simple. You can write tests and benchmarks in your comments, run them from the command line and auto-generate documentation with those same tests executing right in the browser.

Introduction
------------

Here, let's play Show Don't Tell. We'll start with something simple: a function that checks whether a value is an integer. We'll define the function in a file called **numbers.js** and write out a bunch of example cases in a block comment:

```javascript
/**
 * @private
 * @examples
 * isInteger(5)     // => true
 * isInteger(5.0)   // => true
 * isInteger(-5)    // => true
 * isInteger(3.14)  // => false
 * isInteger('foo') // => false
 * isInteger(NaN)   // => false
 */
function isInteger(x) {
  return x === Math.floor(x);
}
```

Notice the `@private` tag. This is necessary because we're doing nothing to expose this function (i.e., there's no `module.exports` for Node); so it lets Autodoc know: "Hey, this function is not exposed publicly but I want to test it anyway."

Now, without doing anything else, right away we can use Autodoc to test this function against our examples.

    $ autodoc --test --verbose numbers.js

    [private]

        isInteger
            isInteger(5) => true
            isInteger(5.0) => true
            isInteger(-5) => true
            isInteger(3.14) => false
            isInteger('foo') => false
            isInteger(NaN) => false

    Finished in 0.006 seconds
    6 tests, 6 assertions, 0 failures, 0 skipped

OK, sweet. Now let's say we've just thought up another possible implementation. Let's add that to the file:

```javascript
/**
 * @private
 * @benchmarks
 * isInteger(123456789)     // using Math.floor
 * isIntegerLike(123456789) // using RegExp: ^\d+$
 */
function isIntegerLike(x) {
  return (/^\d+$/).test(String(x));
}
```

Now we can also use Autodoc to quickly race these two implementations against one another:

    $ autodoc --perf numbers.js

    isIntegerLike - using Math.floor   x 112  - 2,157,257.568 ops/second
    isIntegerLike - using RegExp: ^d+$ x 104  - 1,820,548.837 ops/second

    Results:

    | Function      | Benchmark          | Ops/second    |
    ------------------------------------------------------
    | isIntegerLike | using Math.floor   | 2,157,257.568 |
    | isIntegerLike | using RegExp: ^d+$ | 1,820,548.837 |

All righty. Let's say we're completely delusional and actually think this library is useful, so we want to share it with the world. We'll add some simple comments describing what each function does, remove the `@private` tags, and wrap the library up in a semi-reasonable way so it runs in either Node or the browser:

```javascript
/**
 * Just some simple number helpers.
 */
(function(Numbers) {

  /**
   * Checks if a number is an integer. Returns false for anything that isn't an
   * integer, including non-numbers.
   *
   * [...]
   */
  Numbers.isInteger = function(x) { /*...*/ };

  /**
   * Checks if a value looks like an integer. Returns true for both integers and
   * strings that represent integers, false for everything else.
   *
   * [...]
   */
  Numbers.isIntegerLike = function(x) { /*...*/ };

}(typeof module === 'object' ? (module.exports = {}) : (this.Numbers = {})));
```

Now that we've done that, let's go ahead and generate our API documentation. This is Autodoc's bread and butter:

    $ autodoc numbers.js

![Docs screenshot](http://breakneck.danieltao.com/images/docs_screenshot.png)

Would you look at that? API docs, *with* the our examples and benchmarks, executing directly in the browser.

If you're sold that this is a good idea, read on for more details on how Autodoc works and how to use it. **Keep in mind that as this is a brand new project, a lot of this stuff is in flux and I'm frequently changing my mind about things.** So if you plan to use this tool right now you'll want to follow the project closely on GitHub to find out when things break, new features get added, etc.

Conventions
-----------

By default, the `autodoc` command will put the following files in the **autodoc** folder:

    autodoc/
        index.html
        docs.js
        docs.css

You can change which folder you want this stuff dumped to w/ the `-o` or `--output` option.

To spice up your API docs with some custom JavaScript, add a file called **doc_helper.js** to the output folder. Autodoc will automatically detect it there and add a `<script>` tag referencing it to the resulting HTML. You can add other arbitrary JavaScript files by providing a comma-delimited list via the `--javascripts` option.

You can create your own **docs.css** file, or modify the one Autodoc puts there, and Autodoc will not overwrite it. You can also specify your own template (currently only Mustache templates are supported, though that will change) using the `--template` option. This way you can give your docs a unique look. (Note that if you're using your own template, some other features are not guaranteed to work; e.g., Autodoc will not magically know where to add `<script>` tags linking to **doc_helper.js** or other custom JavaScript files. You'd need to put those into your custom template yourself.)

Other Options
-------------

To only generate documentation for functions with certain tags, you can specify those tags as a comma-separated list via the `--tags` option. By default, *if* you don't specify any tags, but your comments do include some functions with the `@public` tag, then Autodoc will assume you only want those public methods included in the API docs. Otherwise it will include everything in the docs. (Think this is a stupid idea? [Let me know!](https://github.com/dtao/autodoc/issues/new))

If you just want to run specs for certain methods you can use the `--grep` option, which does just what you think.

To see a list of all available options, run `autodoc --help`. I'm pretty sure the list will change pretty regularly in the near term, so that's a better resource for now than this README will be.

### Documentation

API docs will be generated based on the comments above each function. This includes information from `@param` and `@returns` tags. See [JsDoc](http://usejsdoc.org/) for details. You can use [Markdown](http://daringfireball.net/projects/markdown/) to format your comments.

If you have a comment at the top of your file with the `@name` tag, Autodoc will use that as the name of your library. You can use the `@fileOverview` tag to provide a high-level description. Otherwise, Autodoc will just use the topmost comment block in the file, whatever it is.

### Specs

Use the `@examples` tag to define specs above any function:

```javascript
/**
 * @examples
 * trim(' foo') // => 'foo'
 * trim('foo ') // => 'foo'
 */
 function trim(str) { return str.replace(/^\s+|\s+$/g, ''); }
```

#### Default Handlers

Autodoc supports the following syntaxes for defining assertions:

    // The result of calling foo() should be 'bar'
    foo() // => 'bar'

    // The result of calling foo() should be an instance of Foo
    foo() // instanceof Foo

    // After calling foo(), x should equal 5
    foo() // x == 5

    // After calling foo(), x should NOT equal 5
    foo() // x != 5

    // Calling foo() should throw an exception
    foo() // throws

    // Calling foo(callback) should in turn call callback() exactly once
    foo(callback) // calls callback 1 time

    // Calling foo(callback) should call callback() twice asynchronously
    foo(callback) // calls callback 2 times asynchronously

#### Custom Handlers

You can provide custom handlers if you need to do something a little more advanced. To do this, add a file called **handlers.js** to your output folder (whatever you specified with `-o`, or **autodoc** by default) and in it define an `exampleHandlers` array that looks like this:

```javascript
this.exampleHandlers = [
  {
    pattern: /pattern 1/,
    template: 'template1'
  },
  {
    pattern: /pattern 2/,
    template: 'template2'
  },
  ...
];
```

For every example in your comments, the expected output (the part to the right of `// =>`) will first be checked against all of your custom handlers (in order) to see if there's a match.

The `template` property should name a Mustache template file in your output folder following the naming convention "_[template name].js.mustache" (so the example above would require two files, "_template1.js.mustache" and "_template2.js.mustache"). The data passed to the template property will include the properties `{ actual, actualEscaped, expected, expectedEscaped, match }`.

- `actual`: The literal string taken directly from the comment on the left of the `// =>`
- `actualEscaped`: An escaped version of `actual` suitable for, e.g., putting inside a JavaScript string in your template
- `expected`: The literal string taken from the comment on the right side of the `// =>`
- `expectedEscaped`: Same as `actualEscaped`, but for `expected`
- `match`: The match data captured by the `pattern` property of your custom handler

For even *more* control, you can also add a `data` function to your handler which accepts the match object from your pattern and returns any arbitrary data:

```javascript
this.exampleHandlers = [
  {
    pattern: /advanced pattern/,
    template: 'advanced',
    data: function(match) {
      return {
        foo: match[1],
        bar: match[2]
      };
    }
  }
];
```

In this case the output from your function will be passed into your template instead of `match`.

### Benchmarks

Use the `@benchmarks` tag to specify cases you want to profile for performance. The format is similar to `@examples`:

```javascript
/**
 * @benchmarks
 * doSomething(shortString)  // short string
 * doSomething(mediumString) // medium string
 * doSomething(longString)   // long string
 */
```
