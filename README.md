# Tiny Validator (for v4 JSON Schema)

[![Build Status](https://secure.travis-ci.org/geraintluff/tv4.png?branch=master)](http://travis-ci.org/geraintluff/tv4) [![Dependency Status](https://gemnasium.com/geraintluff/tv4.png)](https://gemnasium.com/geraintluff/tv4) [![NPM version](https://badge.fury.io/js/tv4.png)](http://badge.fury.io/js/tv4)

Use [json-schema](http://json-schema.org/) [draft v4](http://json-schema.org/latest/json-schema-core.html) to validate simple values and complex objects using a rich [validation vocabulary](http://json-schema.org/latest/json-schema-validation.html) ([examples](http://json-schema.org/examples.html)).

There is support for `$ref` with JSON Pointer fragment paths (```other-schema.json#/properties/myKey```).

## Usage 1: Simple validation

```javascript
var valid = tv4.validate(data, schema);
```

If validation returns ```false```, then an explanation of why validation failed can be found in ```tv4.error```.

The error object will look something like:
```json
{
    "code": 0,
    "message": "Invalid type: string",
    "dataPath": "/intKey",
    "schemaKey": "/properties/intKey/type"
}
```

The `"code"` property will refer to one of the values in `tv4.errorCodes` - in this case, `tv4.errorCodes.INVALID_TYPE`.

To enable external schema to be referenced, you use:
```javascript
tv4.addSchema(url, schema);
```

If schemas are referenced (```$ref```) but not known, then validation will return ```true``` and the missing schema(s) will be listed in ```tv4.missing```. For more info see the API documentation below.

## Usage 2: Multi-threaded validation

Storing the error and missing schemas does not work well in multi-threaded environments, so there is an alternative syntax:

```javascript
var result = tv4.validateResult(data, schema);
```

The result will look something like:
```json
{
    "valid": false,
    "error": {...},
    "missing": [...]
}
```

## Usage 3: Multiple errors

Normally, `tv4` stops when it encounters the first validation error.  However, you can collect an array of validation errors using:

```javascript
var result = tv4.validateMultiple(data, schema);
```

The result will look something like:
```json
{
    "valid": false,
    "errors": [
        {...},
        ...
    ],
    "missing": [...]
}
```

## Asynchronous validation

Support for asynchronous validation (where missing schemas are fetched) can be added by including an extra JavaScript file.  Currently, the only version requires jQuery (`tv4.async-jquery.js`), but the code is very short and should be fairly easy to modify for other libraries (such as MooTools).

Usage:

```javascript
tv4.validate(data, schema, function (isValid, validationError) { ... });
```

`validationFailure` is simply taken from `tv4.error`.

## Cyclical JavaScript objects

While they don't occur in proper JSON, JavaScript does support self-referencing objects. Any of the above calls support an optional final argument, checkRecursive. If true, tv4 will handle self-referencing objects properly - this slows down validation slightly, but that's better than a hanging script.

```javascript
var a = {};
var b = { a: a };
a.b = b;
var aSchema = { properties: { b: { $ref: 'bSchema' }}};
var bSchema = { properties: { a: { $ref: 'aSchema' }}};
tv4.addSchema('aSchema', aSchema);
tv4.addSchema('bSchema', bSchema);
tv4.validate(a, aSchema, true); // If the final checkRecursive argument were missing, this would throw a "too much recursion" error.
tv4.validate(a, schema, asynchronousFunction, true); // Works with asynchronous validation.
tv4.validateResult(data, aSchema, true); // Also multi-threaded and multiple error validation.
tv4.validateMultiple(data, aSchema, true);
```

## API

There are additional api commands available for more complex use-cases:

##### addSchema(uri, schema)
Pre-register a schema for reference by other schema and synchronous validation.

````js
tv4.addSchema('http://example.com/schema', { ... });
````

* `uri` the uri to identify this schema.
* `schema` the schema object.

Schemas that have their `id` property set can be added directly.

````js
tv4.addSchema({ ... });
````

##### getSchema(uri)

Return a schema from the cache.

* `uri` the uri of the schema (may contain a `#` fragment)

````js
var schema = tv4.getSchema('http://example.com/schema');
````

##### getSchemaMap()

Return a shallow copy of the schema cache, mapping schema document URIs to schema objects.

````
var map = tv4.getSchemaMap();

var schema = map[uri];
````

##### getSchemaUris(filter)

Return an Array with known schema document URIs.

* `filter` optional RegExp to filter URIs

````
var arr = tv4.getSchemaUris();

// optional filter using a RegExp
var arr = tv4.getSchemaUris(/^https?://example.com/);
````

##### getMissingUris(filter)

Return an Array with schema document URIs that are used as `$ref` in known schemas but which currently have no associated schema data.

Use this in combination with `tv4.addSchema(uri, schema)` to preload the cache for complete synchronous validation with.

* `filter` optional RegExp to filter URIs

````
var arr = tv4.getMissingUris();

// optional filter using a RegExp
var arr = tv4.getMissingUris(/^https?://example.com/);
````

##### dropSchemas()

Drop all known schema document URIs from the cache.

````
tv4.dropSchemas();
````

##### freshApi()

Return a new tv4 instance with no shared state.

````
var otherTV4 = tv4.freshApi();
````

##### reset()

Manually reset validation status from the simple `tv4.validate(data, schema)`. Although tv4 will self reset on each validation there are some implementation scenarios where this is useful.

````
tv4.reset();
````

##### language(code)

Select the language map used for reporting.

* `code` is a langauge code, like `'en'` or `'en-gb'`

````
tv4.language('en-gb');
````

##### addLanguage(code, map)

Add a new language map for selection by `tv4.language(code)`

* `code` is new language code
* `map` is an object mapping error IDs or constant names (e.g. `103` or `"NUMBER_MAXIMUM"`) to language strings.

````
tv4.addLanguage('fr', { ... });

// select for use
tv4.language('fr')
````
## Demos

### Basic usage
<div class="content" markdown="1">
<a href="javascript:runDemo('demo1');">run demo</a>
<pre class="code" id="demo1">
var schema = {
	"items": {
		"type": "boolean"
	}
};
var data1 = [true, false];
var data2 = [true, 123];

alert("data 1: " + tv4.validate(data1, schema)); // true
alert("data 2: " + tv4.validate(data2, schema)); // false
alert("data 2 error: " + JSON.stringify(tv4.error, null, 4));
</pre>
</div>

### Use of <code>$ref</code>
<div class="content">
<a href="javascript:runDemo('demo2');">run demo</a>
<pre class="code" id="demo2">
var schema = {
	"type": "array",
	"items": {"$ref": "#"}
};
var data1 = [[], [[]]];
var data2 = [[], [true, []]];

alert("data 1: " + tv4.validate(data1, schema)); // true
alert("data 2: " + tv4.validate(data2, schema)); // false
</pre>
</div>

### Missing schema
<div class="content">
<a href="javascript:runDemo('demo3');">run demo</a>
<pre class="code" id="demo3">
var schema = {
	"type": "array",
	"items": {"$ref": "http://example.com/schema"}
};
var data = [1, 2, 3];

alert("Valid: " + tv4.validate(data, schema)); // true
alert("Missing schemas: " + JSON.stringify(tv4.missing));
</pre>
</div>

### Referencing remote schema
<div class="content">
<a href="javascript:runDemo('demo4');">run demo</a>
<pre class="code" id="demo4">
tv4.addSchema("http://example.com/schema", {
	"definitions": {
		"arrayItem": {"type": "boolean"}
	}
});
var schema = {
	"type": "array",
	"items": {"$ref": "http://example.com/schema#/definitions/arrayItem"}
};
var data1 = [true, false, true];
var data2 = [1, 2, 3];

alert("data 1: " + tv4.validate(data1, schema)); // true
alert("data 2: " + tv4.validate(data2, schema)); // false
</pre>
</div>

## Supported platforms

* Node.js
* All modern browsers
* IE >= 7

## Installation

You can manually download [`tv4.js`](https://raw.github.com/geraintluff/tv4/master/tv4.js) or the minified [`tv4.min.js`](https://raw.github.com/geraintluff/tv4/master/tv4.min.js) and include it in your html to create the global `tv4` variable.

Alternately use it as a CommonJS module:

````js
var tv4 = require('tv4').tv4;
````

#### npm

````
$ npm install tv4
````

#### bower

````
$ bower install tv4
````

#### component.io

````
$ component install geraintluff/tv4
````

## Build and test

You can rebuild and run the node and browser tests using node.js and [grunt](http://http://gruntjs.com/):

Make sure you have the global grunt cli command:
````
$ npm install grunt-cli -g
````

Clone the git repos, open a shell in the root folder and install the development dependencies:

````
$ npm install
````

Rebuild and run the tests:
````
$ grunt
````

It will run a build and display one Spec-style report for the node.js and two Dot-style reports for both the plain and minified browser tests (via phantomJS). You can also use your own browser to manually run the suites by opening [`test/index.html`](http://geraintluff.github.io/tv4/test/index.html) and [`test/index-min.html`](http://geraintluff.github.io/tv4/test/index-min.html).

## Contributing

Pull-requests for fixes and expansions are welcome. Edit the partial files in `/source` and add your tests in a suitable suite or folder under `/test/tests` and run `grunt` to rebuild and run the test suite. Try to maintain an idiomatic coding style and add tests for any new features. It is recommend to discuss big changes in an Issue.

## Packages using tv4

* [chai-json-schema](http://chaijs.com/plugins/chai-json-schema) is a [Chai Assertion Library](http://chaijs.com) plugin to assert values against json-schema.
* [grunt-tv4](http://www.github.com/Bartvds/grunt-tv4) is a plugin for [grunt](http://http://gruntjs.com/) that uses tv4 to bulk validate json files.

## License

The code is available as "public domain", meaning that it is completely free to use, without any restrictions at all.  Read the full license [here](http://geraintluff.github.com/tv4/LICENSE.txt).

It's also available under an [MIT license](http://jsonary.com/LICENSE.txt).
