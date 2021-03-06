# clue/ndjson-react [![Build Status](https://travis-ci.org/clue/php-ndjson-react.svg?branch=master)](https://travis-ci.org/clue/php-ndjson-react)

Streaming newline delimited JSON ([NDJSON](http://ndjson.org/)) parser and encoder, built on top of React PHP

**Table of Contents**

* [Usage](#usage)
  * [Decoder](#decoder)
  * [Encoder](#encoder)
* [Install](#install)
* [Tests](#tests)
* [License](#license)
* [More](#more)

## Usage

### Decoder

The `Decoder` (parser) class can be used to make sure you only get back
complete, valid JSON elements when reading from a stream.
It wraps a given
[`ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface)
and exposes its data through the same interface, but emits the JSON elements
as parsed values instead of just chunks of strings:

```
{"name":"test","active":true}\r\n
"hello w\u00f6rld"\r\n
```
```php
$stdin = new Stream(STDIN, $loop);

$stream = new Decoder($stdin);

$stream->on('data', function ($data) {
    // data is a parsed element form the JSON stream
    // line 1: $data = (object)array('name' => 'test', 'active' => true);
    // line 2: $data = "hello wörld";
    var_dump($data);
});
```

React's streams emit chunks of data strings and make no assumption about their lengths.
These chunks do not necessarily represent complete JSON elements, as an
element may be broken up into multiple chunks.
This class reassembles these elements by buffering incomplete ones.

The `Decoder` supports the same parameters as the underlying
[`json_decode()`](http://php.net/json_decode) function.
This means that, by default, JSON objects will be emitted as a `stdClass`.
This behavior can be controlled through the optional constructor parameters:

```php
$stream = new Decoder($stdin, true);

$stream->on('data', function ($data) {
    // JSON objects will be emitted as assoc arrays now
});
```

If the underlying stream emits an `error` event or the plain stream contains
any data that does not represent a valid NDJson stream,
it will emit an `error` event and then `close` the input stream:

```php
$stream->on('error', function (Exception $error) {
    // an error occured, stream will close next
});
```

If the underlying stream emits an `end` event, it will flush any incomplete
data from the buffer, thus either possibly emitting a final `data` event
followed by an `end` event on success or an `error` event for
incomplete/invalid JSON data as above:

```php
$stream->on('end', function () {
    // stream successfully ended, stream will close next
});
```

If either the underlying stream or the `Decoder` is closed, it will forward
the `close` event:

```php
$stream->on('close', function () {
    // stream closed
    // possibly after an "end" event or due to an "error" event
});
```

The `close(): void` method can be used to explicitly close the `Decoder` and
its underlying stream:

```php
$stream->close();
```

The `pipe(WritableStreamInterface $dest, array $options = array(): WritableStreamInterface`
method can be used to forward all data to the given destination stream.
Please note that the `Decoder` emits decoded/parsed data events, while many
(most?) writable streams expect only data chunks:

```php
$stream->pipe($logger);
```

For more details, see React's
[`ReadableStreamInterface`](https://github.com/reactphp/stream#readablestreaminterface).

### Encoder

The `Encoder` (serializer) class can be used to make sure anything you write to
a stream ends up as valid JSON elements in the resulting NDJSON stream.
It wraps a given
[`WritableStreamInterface`](https://github.com/reactphp/stream#writablestreaminterface)
and accepts its data through the same interface, but handles any data as complete
JSON elements instead of just chunks of strings:

```php
$stdout = new Stream(STDOUT, $loop);

$stream = new Encoder($stdout);

$stream->write(array('name' => 'test', 'active' => true));
$stream->write('hello wörld');
```
```
{"name":"test","active":true}\r\n
"hello w\u00f6rld"\r\n
```

The `Encoder` supports the same parameters as the underlying
[`json_encode()`](http://php.net/json_encode) function.
This means that, by default, unicode characters will be escaped in the output.
This behavior can be controlled through the optional constructor parameters:

```php
$stream = new Encoder($stdout, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE);

$stream->write('hello wörld');
```
```
"hello wörld"\r\n
```

Note that trying to pass the `JSON_PRETTY_PRINT` option will yield an
`InvalidArgumentException` because it is not compatible with NDJSON.

If the underlying stream emits an `error` event or the given datacontains
any data that can not be represented as a valid NDJSON stream,
it will emit an `error` event and then `close` the input stream:

```php
$stream->on('error', function (Exception $error) {
    // an error occured, stream will close next
});
```

If either the underlying stream or the `Encoder` is closed, it will forward
the `close` event:

```php
$stream->on('close', function () {
    // stream closed
    // possibly after an "end" event or due to an "error" event
});
```

The `end(mixed $data = null): void` method can be used to optionally emit
any final data and then soft-close the `Encoder` and its underlying stream:

```php
$stream->end();
```

The `close(): void` method can be used to explicitly close the `Encoder` and
its underlying stream:

```php
$stream->close();
```

For more details, see React's
[`WritableStreamInterface`](https://github.com/reactphp/stream#writablestreaminterface).


## Install

The recommended way to install this library is [through Composer](https://getcomposer.org).
[New to Composer?](https://getcomposer.org/doc/00-intro.md)

This will install the latest supported version:

```bash
$ composer require clue/ndjson-react:^0.1
```

See also the [CHANGELOG](CHANGELOG.md) for details about version upgrades.

## Tests

To run the test suite, you first need to clone this repo and then install all dependencies [through Composer](http://getcomposer.org):

```bash
$ composer install
```

To run the test suite, go to the project root and run:

```bash
$ php vendor/bin/phpunit
```

## License

MIT

## More

* If you want to learn more about processing streams of data, refer to the documentation of
  the underlying [react/stream](https://github.com/reactphp/stream) component.

* If you want to process compressed NDJSON files (`.ndjson.gz` file extension)
  you may want to use [clue/zlib-react](https://github.com/clue/php-zlib-react)
  on the compressed input stream before passing the decompressed stream to the NDJSON decoder.

* If you want to create compressed NDJSON files (`.ndjson.gz` file extension)
  you may want to use [clue/zlib-react](https://github.com/clue/php-zlib-react)
  on the resulting NDJSON encoder output stream before passing the compressed
  stream to the file output stream.
