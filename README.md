<a name="roarr"></a>
# Roarr

[![GitSpo Mentions](https://gitspo.com/badges/mentions/gajus/roarr?style=flat-square)](https://gitspo.com/mentions/gajus/roarr)
[![Travis build status](http://img.shields.io/travis/gajus/roarr/master.svg?style=flat-square)](https://travis-ci.org/gajus/roarr)
[![Coveralls](https://img.shields.io/coveralls/gajus/roarr.svg?style=flat-square)](https://coveralls.io/github/gajus/roarr)
[![NPM version](http://img.shields.io/npm/v/roarr.svg?style=flat-square)](https://www.npmjs.org/package/roarr)
[![Canonical Code Style](https://img.shields.io/badge/code%20style-canonical-blue.svg?style=flat-square)](https://github.com/gajus/canonical)
[![Twitter Follow](https://img.shields.io/twitter/follow/kuizinas.svg?style=social&label=Follow)](https://twitter.com/kuizinas)

JSON logger for Node.js and browser.

* [Roarr](#roarr)
    * [Motivation](#roarr-motivation)
    * [Usage](#roarr-usage)
        * [Filtering logs](#roarr-usage-filtering-logs)
        * [jq primer](#roarr-usage-jq-primer)
    * [Log message format](#roarr-log-message-format)
    * [API](#roarr-api)
        * [`child`](#roarr-api-child)
        * [`getContext`](#roarr-api-getcontext)
        * [`trace`](#roarr-api-trace)
        * [`debug`](#roarr-api-debug)
        * [`info`](#roarr-api-info)
        * [`warn`](#roarr-api-warn)
        * [`error`](#roarr-api-error)
        * [`fatal`](#roarr-api-fatal)
    * [Middlewares](#roarr-middlewares)
    * [CLI program](#roarr-cli-program)
    * [Transports](#roarr-transports)
    * [Environment variables](#roarr-environment-variables)
    * [Conventions](#roarr-conventions)
        * [Context property names](#roarr-conventions-context-property-names)
        * [Using Roarr in an application](#roarr-conventions-using-roarr-in-an-application)
    * [Recipes](#roarr-recipes)
        * [Logging errors](#roarr-recipes-logging-errors)
        * [Using with Elasticsearch](#roarr-recipes-using-with-elasticsearch)
        * [Using with Scalyr](#roarr-recipes-using-with-scalyr)
        * [Documenting use of Roarr](#roarr-recipes-documenting-use-of-roarr)


<a name="roarr-motivation"></a>
## Motivation

For a long time I have been a big fan of using [`debug`](https://github.com/visionmedia/debug). `debug` is simple to use, works in Node.js and browser, does not require configuration and it is fast. However, problems arise when you need to parse logs. Anything but one-line text messages cannot be parsed in a safe way.

To log structured data, I have been using [Winston](https://github.com/winstonjs/winston) and [Bunyan](https://github.com/trentm/node-bunyan). These packages are great for application-level logging. I have preferred Bunyan because of the [Bunyan CLI program](https://github.com/trentm/node-bunyan#cli-usage) used to pretty-print logs. However, these packages require program-level configuration – when constructing an instance of a logger, you need to define the transport and the log-level. This makes them unsuitable for use in code designed to be consumed by other applications.

Then there is [pino](https://github.com/pinojs/pino). pino is fast JSON logger, it has CLI program equivalent to Bunyan, it decouples transports, and it has sane default configuration. Unfortunately, you still need to instantiate logger instance at the application-level. This makes it more suitable for application-level logging just like Winston and Bunyan.

I needed a logger that:

* Does not block the event cycle (=fast).
* Does not require initialisation.
* Produces structured data.
* [Decouples transports](#transports).
* Has a [CLI program](#cli-program).
* Works in Node.js and browser.
* Configurable using environment variables.

In other words,

* a logger that I can use in an application code and in dependencies.
* a logger that allows to correlate logs between the main application code and the dependency code.
* a logger that works well with transports in external processes.

Roarr is this logger.

<a name="roarr-usage"></a>
## Usage

Roarr logging is disabled by default. To enable logging, you must start program with an environment variable `ROARR_LOG` set to `true`, e.g.

```bash
ROARR_LOG=true node ./index.js

```

```js
import log from 'roarr';

log('foo');

log('bar %s', 'baz');

const debug = log.child({
  logLevel: 10
});

debug('qux');

debug({
  quuz: 'corge'
}, 'quux');

```

Produces output:

```
{"context":{},"message":"foo","sequence":0,"time":1506776210000,"version":"1.0.0"}
{"context":{},"message":"bar baz","sequence":1,"time":1506776210000,"version":"1.0.0"}
{"context":{"logLevel":10},"message":"qux","sequence":2,"time":1506776210000,"version":"1.0.0"}
{"context":{"logLevel":10,"quuz":"corge"},"sequence":3,"message":"quux","time":1506776210000,"version":"1.0.0"}

```

<a name="roarr-usage-filtering-logs"></a>
### Filtering logs

Roarr is designed to print all or none logs (refer to the [`ROARR_LOG` environment variable](#environment-variables) documentation).

To filter logs you need to use [`roarr filter` CLI program](#filter-program) or a JSON processor such as [jq](https://stedolan.github.io/jq/).

<a name="roarr-usage-jq-primer"></a>
### jq primer

`jq` allows you to filter JSON messages using [`select(boolean_expression)`](https://stedolan.github.io/jq/manual/#select(boolean_expression)), e.g.

```bash
ROARR_LOG=true node ./index.js | jq 'select(.context.logLevel > 40)'

```

Combine it with `roarr pretty-print` to pretty-print a subset of the logs:

```bash
ROARR_LOG=true node ./index.js | jq -cM 'select(.context.logLevel > 40)'

```

(Notice the use of `-cM` parameters to disable JSON colorization and formatting.)

If your application outputs non-JSON output, jq will fail with an error similar to:

```
parse error: Invalid numeric literal at line 1, column 5
Error: write EPIPE
    at _errnoException (util.js:1031:13)
    at WriteWrap.afterWrite (net.js:873:14)

```

To ignore the non-JSON output, use jq `-R` flag (raw input) in combination with [`fromjson`](https://stedolan.github.io/jq/manual/#Convertto/fromJSON), e.g.

```bash
ROARR_LOG=true node ./index.js | jq -cRM 'fromjson? | select(.context.logLevel > 40)'

```

For a simplified way of filtering Roarr logs, refer to [`roarr filter` CLI program](#filter-program).

<a name="roarr-log-message-format"></a>
## Log message format

|Property name|Contents|
|---|---|
|`context`|Arbitrary, user-provided structured data. See [context property names](#context-property-names).|
|`message`|User-provided message formatted using [printf](https://en.wikipedia.org/wiki/Printf_format_string).|
|`sequence`|An incremental ID.|
|`time`|Unix timestamp in milliseconds.|
|`version`|Roarr log message format version.|

Example:

```js
{
  "context": {
    "application": "task-runner",
    "hostname": "curiosity.local",
    "instanceId": "01BVBK4ZJQ182ZWF6FK4EC8FEY",
    "taskId": 1
  },
  "message": "starting task ID 1",
  "sequence": 0,
  "time": 1506776210000,
  "version": "1.0.0"
}

```

<a name="roarr-api"></a>
## API

`roarr` package exports a function that accepts the following API:

```js
export type LoggerType =
  (
    context: MessageContextType,
    message: string,
    c?: SprintfArgumentType,
    d?: SprintfArgumentType,
    e?: SprintfArgumentType,
    f?: SprintfArgumentType,
    g?: SprintfArgumentType,
    h?: SprintfArgumentType,
    i?: SprintfArgumentType,
    k?: SprintfArgumentType
  ) => void |
  (
    message: string,
    b?: SprintfArgumentType,
    c?: SprintfArgumentType,
    d?: SprintfArgumentType,
    e?: SprintfArgumentType,
    f?: SprintfArgumentType,
    g?: SprintfArgumentType,
    h?: SprintfArgumentType,
    i?: SprintfArgumentType,
    k?: SprintfArgumentType
  ) => void;

```

To put it into words:

* First parameter can be either a string (message) or an object.
  * If first parameter is an object (context), the second parameter must be a string (message).
* Arguments after the message parameter are used to enable [printf message formatting](https://en.wikipedia.org/wiki/Printf_format_string).
  * Printf arguments must be of a primitive type (`string | number | boolean | null`).
  * There can be up to 9 printf arguments (or 8 if the first parameter is the context object).

Refer to the [Usage documentation](#usage) for common usage examples.

<a name="roarr-api-child"></a>
### <code>child</code>

The `child` function has two signatures:

1. Accepts an object.
2. Accepts a function.

<a name="roarr-api-child-object-parameter"></a>
#### Object parameter

Creates a child logger appending the provided `context` object to the previous logger context.

```js
type ChildType = (context: MessageContextType) => LoggerType;

```

Example:

```js
import log from 'roarr';

const childLog = log.child({
  foo: 'bar'
});

log.debug('foo 1');
childLog.debug('foo 2');

// {"context":{"logLevel":20},"message":"foo 1","sequence":0,"time":1531914529921,"version":"1.0.0"}
// {"context":{"foo":"bar","logLevel":20},"message":"foo 2","sequence":1,"time":1531914529922,"version":"1.0.0"}

```

Refer to [middlewares](#middlewares) documentation for use case examples.

<a name="roarr-api-child-function-parameter"></a>
#### Function parameter

Creates a child logger where every message is intercepted.

```js
type ChildType = (translateMessage: TranslateMessageFunctionType) => LoggerType;

```

Example:

```js
import log from 'roarr';

const childLog = log.child((message) => {
  return {
    ...message,
    message: message.message.replace('foo', 'bar')
  }
});

log.debug('foo 1');
childLog.debug('foo 2');

// {"context":{"logLevel":20},"message":"foo 1","sequence":0,"time":1531914656076,"version":"1.0.0"}
// {"context":{"logLevel":20},"message":"bar 2","sequence":1,"time":1531914656077,"version":"1.0.0"}

```

<a name="roarr-api-getcontext"></a>
### <code>getContext</code>

Returns the current context.

<a name="roarr-api-trace"></a>
### <code>trace</code>
<a name="roarr-api-debug"></a>
### <code>debug</code>
<a name="roarr-api-info"></a>
### <code>info</code>
<a name="roarr-api-warn"></a>
### <code>warn</code>
<a name="roarr-api-error"></a>
### <code>error</code>
<a name="roarr-api-fatal"></a>
### <code>fatal</code>

Convenience methods for logging a message with `logLevel` context property value set to the name of the convenience method, e.g.

```js
import log from 'roarr';

log.trace('foo');
log.debug('foo');
log.info('foo');
log.warn('foo');
log.error('foo');
log.fatal('foo');

```

Produces output:

```
{"context":{"logLevel":10},"message":"foo","sequence":0,"time":1506776210000,"version":"1.0.0"}
{"context":{"logLevel":20},"message":"foo","sequence":1,"time":1506776210000,"version":"1.0.0"}
{"context":{"logLevel":30},"message":"foo","sequence":2,"time":1506776210000,"version":"1.0.0"}
{"context":{"logLevel":40},"message":"foo","sequence":3,"time":1506776210000,"version":"1.0.0"}
{"context":{"logLevel":50},"message":"foo","sequence":4,"time":1506776210000,"version":"1.0.0"}
{"context":{"logLevel":60},"message":"foo","sequence":5,"time":1506776210000,"version":"1.0.0"}

```

<a name="roarr-middlewares"></a>
## Middlewares

Roarr logger supports middlewares implemented as [`child`](#child) message translate functions, e.g.

```js
import log from 'roarr';
import createSerializeErrorMiddleware from '@roarr/middleware-serialize-error';

const childLog = log.child(createSerializeErrorMiddleware());

const error = new Error('foo');

log.debug({error}, 'bar');
childLog.debug({error}, 'bar');

// {"context":{"logLevel":20,"error":{}},"message":"bar","sequence":0,"time":1531918373676,"version":"1.0.0"}
// {"context":{"logLevel":20,"error":{"name":"Error","message":"foo","stack":"[REDACTED]"}},"message":"bar","sequence":1,"time":1531918373678,"version":"1.0.0"}

```

Roarr middlwares enable translation of every bit of information that is used to construct a log message.

The following are the official middlewares:

* [`@roarr/middleware-serialize-error`](https://github.com/gajus/roarr-middleware-serialize-error)

Raise an issue to add your middleware of your own creation.

<a name="roarr-cli-program"></a>
## CLI program

Roarr CLI program provides ability to augment, filter and pretty-print Roarr logs.

![CLI output demo](./.README/cli-output-demo.png)

CLI program has been moved to a separate package [`@roarr/cli`](https://github.com/gajus/roarr-cli).

```bash
npm install @roarr/cli -g

```

Explore all CLI commands and options using `roarr --help` or refer to [`@roarr/cli`](https://github.com/gajus/roarr-cli) documentation.

<a name="roarr-transports"></a>
## Transports

A transport in most logging libraries is something that runs in-process to perform some operation with the finalised log line. For example, a transport might send the log line to a standard syslog server after processing the log line and reformatting it.

Roarr does not support in-process transports.

Roarr does not support in-process transports because Node processes are single threaded processes (ignoring some technical details). Given this restriction, Roarr purposefully offloads handling of the logs to external processes so that the threading capabilities of the OS can be used (or other CPUs).

Depending on your configuration, consider one of the following log transports:

* [Beats](https://www.elastic.co/products/beats) for aggregating at a process level (written in Go).
* [logagent](https://github.com/sematext/logagent-js) for aggregating at a process level (written in JavaScript).
* [Fluentd](https://www.fluentd.org/) for aggregating logs at a container orchestration level (e.g. Kubernetes) (written in Ruby).

<a name="roarr-environment-variables"></a>
## Environment variables

When running the script in a Node.js environment, use environment variables to control `roarr` behaviour.

|Name|Type|Function|Default|
|---|---|---|---|
|`ROARR_LOG`|Boolean|Enables/ disables logging.|`false`|
|`ROARR_STREAM`|`STDOUT`, `STDERR`|Name of the stream where the logs will be written.|`STDOUT`|
|`ROARR_BUFFER_SIZE`|Number|Configures the buffer size. Buffer is used to store messages before printing them to the stdout/ stderr. Recommended buffer size depends on how often program produces logs. Experiment with values 1024, 2048, 4096 and 8192.|`0` (disabled)|

When using `ROARR_STREAM=STDERR`, use [`3>&1 1>&2 2>&3 3>&-`](https://stackoverflow.com/a/2381643/368691) to pipe stderr output.

<a name="roarr-conventions"></a>
## Conventions

<a name="roarr-conventions-context-property-names"></a>
### Context property names

Roarr does not have reserved context property names. However, I encourage use of the following conventions:

|Context property name|Use case|
|---|---|
|`application`|Name of the application (do not use in code intended for distribution; see `package` property instead).|
|`hostname`|Machine hostname. See `roarr augment --append-hostname` option.|
|`instanceId`|Unique instance ID. Used to distinguish log source in high-concurrency environments. See `roarr augment --append-instance-id` option.|
|`logLevel`|A numeric value indicating the [log level](#log-levels). See [API](#api) for the build-in loggers with a pre-set log-level.|
|`namespace`|Namespace within a package, e.g. function name. Treat the same way that you would construct namespaces when using the [`debug`](https://github.com/visionmedia/debug) package.|
|`package`|Name of the package.|

The `roarr pretty-print` [CLI program](#cli-program) is using the context property names suggested in the conventions to pretty-print the logs for the developer inspection purposes.

<a name="roarr-conventions-context-property-names-log-levels"></a>
#### Log levels

The `roarr pretty-print` [CLI program](#cli-program) translates `logLevel` values to the following human-readable names:

|`logLevel`|Human-readable name|
|---|---|
|10|TRACE|
|20|DEBUG|
|30|INFO|
|40|WARN|
|50|ERROR|
|60|FATAL|

<a name="roarr-conventions-using-roarr-in-an-application"></a>
### Using Roarr in an application

To avoid code duplication, you can use a singleton pattern to export a logger instance with predefined context properties (e.g. describing the application).

I recommend to create a file `Logger.js` in the project directory. Use this file to create an child instance of Roarr with context parameters describing the project and the initialisation instance, e.g.

```js
/**
 * @file Example contents of a Logger.js file.
 */

import log from 'roarr';

const Logger = log.child({
  // .foo property is going to appear only in the logs that are created using
  // the current instance of a Roarr logger.
  foo: 'bar'
});

export default Logger;

```

Roarr does not have reserved context property names. However, I encourage use of the conventions. The `roarr pretty-print` [CLI program](#cli-program) is using the context property names suggested in the [conventions](#conventions) to pretty-print the logs for the developer inspection purposes.

<a name="roarr-recipes"></a>
## Recipes

<a name="roarr-recipes-logging-errors"></a>
### Logging errors

This is not specific to Roarr – this suggestion applies to any kind of logging.

If you want to include an instance of [`Error`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error) in the context, you must serialize the error.

The least-error prone way to do this is to use an existing library, e.g. [`serialize-error`](https://www.npmjs.com/package/serialize-error).

```js
import log from 'roarr';
import serializeError from 'serialize-error';

// [..]

send((error, result) => {
  if (error) {
    log.error({
      error: serializeError(error)
    }, 'message not sent due to a remote error');

    return;
  }

  // [..]
});

```

Without using serialisation, your errors will be logged without the error name and stack trace.

<a name="roarr-recipes-using-with-elasticsearch"></a>
### Using with Elasticsearch

If you are using [Elasticsearch](https://www.elastic.co/products/elasticsearch), you will want to create an [index template](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html).

The following serves as the ground work for the index template. It includes the main Roarr log message properties (context, message, time) and the context properties suggested in the [conventions](#conventions).

```json
{
  "mappings": {
    "log_message": {
      "_source": {
        "enabled": true
      },
      "dynamic": "strict",
      "properties": {
        "context": {
          "dynamic": true,
          "properties": {
            "application": {
              "type": "keyword"
            },
            "hostname": {
              "type": "keyword"
            },
            "instanceId": {
              "type": "keyword"
            },
            "logLevel": {
              "type": "integer"
            },
            "namespace": {
              "type": "text"
            },
            "package": {
              "type": "text"
            }
          }
        },
        "message": {
          "type": "text"
        },
        "time": {
          "format": "epoch_millis",
          "type": "date"
        }
      }
    }
  },
  "template": "logstash-*"
}

```

<a name="roarr-recipes-using-with-scalyr"></a>
### Using with Scalyr

If you are using [Scalyr](https://www.scalyr.com/), you will want to create a custom parser `RoarrLogger`:

```js
{
  patterns: {
    tsPattern: "\\w{3},\\s\\d{2}\\s\\w{3}\\s\\d{4}\\s[\\d:]+",
    tsPattern_8601: "\\d{4}-\\d{2}-\\d{2}T[\\d:.]+Z"
  }
  formats: [
    {format: "${parse=json}$"},
    {format: ".*\"time\":$timestamp=number$,.*"},
    {format: "$timestamp=tsPattern$ GMT $detail$"},
    {format: "$timestamp=tsPattern_8601$ $detail$"}
  ]
}

```

and configure the individual programs to use `RoarrLogger`. In case of Kubernetes, this means adding a `log.config.scalyr.com/attributes.parser: RoarrLogger` annotation to the associated deployment, pod or container.

<a name="roarr-recipes-documenting-use-of-roarr"></a>
### Documenting use of Roarr

If your package is using Roarr, include instructions to README.md describing how to enable logging, e.g.

```markdown
## Logging

This package is using [`roarr`](https://www.npmjs.com/package/roarr) logger to log the program's state.

Export `ROARR_LOG=true` environment variable to enable log printing to stdout.

Use [`roarr-cli`](https://github.com/gajus/roarr-cli) program to pretty-print the logs.

```
