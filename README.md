# Ponos

[![travis]](https://travis-ci.org/Runnable/ponos)
[![coveralls]](https://coveralls.io/github/Runnable/ponos?branch=master)
[![dependencies]](https://david-dm.org/Runnable/ponos)
[![devdependencies]](https://david-dm.org/Runnable/ponos#info=devDependencies)
[![codeclimate]](https://codeclimate.com/github/Runnable/ponos)

Documentation is available at [runnable.github.io/ponos][documentation]

An opinionated queue based worker server for node.

For ease of use we provide options to set the host, port, username, and password to the RabbitMQ server. If not present in options, the server will attempt to use the following environment variables and final defaults:

options         | environment         | default
----------------|---------------------|--------------
`opts.hostname` | `RABBITMQ_HOSTNAME` | `'localhost'`
`opts.port`     | `RABBITMQ_PORT`     | `'5672'`
`opts.username` | `RABBITMQ_USERNAME` | `'guest'`
`opts.password` | `RABBITMQ_PASSWORD` | `'guest'`
`opts.log`      | *N/A*               | Basic [bunyan](https://github.com/trentm/node-bunyan) instance with `stdout` stream (for logging)
`opts.errorCat` | *N/A*               | Basic [error-cat](https://github.com/runnable/error-cat) instance (for rollbar error reporting)


Other options for Ponos are as follows:

Environment              | default | description
-------------------------|---------|------------
`WORKER_MAX_RETRY_DELAY` | `0`     | The maximum time, in milliseconds, that the worker will wait before retrying a task. The timeout will exponentially increase from `MIN_RETRY_DELAY` to `MAX_RETRY_DELAY` if the latter is set higher than the former. If this value is not set, the worker will not exponentially back off.
`WORKER_MIN_RETRY_DELAY` | `1`     | Time, in milliseconds, the worker will wait at minimum will wait before retrying a task.
`WORKER_TIMEOUT`         | `0`     | Timeout, in milliseconds, at which the worker task will be retried.


## Usage

From a high level, Ponos is used to create a worker server that responds to jobs provided from RabbitMQ. The user defines handlers for each queue's jobs that are invoked by Ponos.

Ponos has built in support for retrying and catching specific errors, which are described below.

## Workers

Workers need to be defined as a function that takes a Object `job` an returns a promise. For example:

```javascript
function myWorker (job) {
  return Promise.resolve()
    .then(() => {
      return doSomeWork(job)
    })
}
```

This worker takes the `job`, does work with it, and returns the result. Since (in theory) this does not throw any errors, the worker will see this resolution and acknowledge the job.

### Interacting with RabbitMQ and Ponos Server

Workers optionally can take a second argument that allows them to interact with the Ponos server and RabbitMQ topology. For example, if topic exchange queues needed to be created on the fly and subscribed to, we could do the following:

```javascript
function workerFoo (job) {
  console.log('foo received a job')
  return Promise.resolve()
}

function workerBar (job, ponos) {
  console.log('bar received a job')
  // subscribe to the exchange with the routing key
  return ponos.subscribe({
    exchange: 'hello-exchange',
    routingKey: '#.foo',
    handler: workerFoo
  })
    .then(() => {
      // trigger a consume, starting the new queue
      return ponos.consume()
    })
    .then(() => {
      console.log('new route queue created and subscribed')
    })
}

// create a server that subscribes to the #.bar route
const server = new Ponos.Server({
  exchanges: [{
    exchange: 'hello-exchange',
    routingKey: '#.bar',
    handler: workerBar
  }]
})
// start the server
server.start()
```

Initially, if we publish a job to `hello-exchange` with the key `a.foo`, we will not receive any message that `foo received a job`. When we publish a job that triggers `workerBar`, it has Ponos subscribe to and consume on the exchange with a new routing key. Now when we publish back with `a.foo`, we see the log that `foo received a job`.

A picture says a thousand words, so check out `topic-exchanges-on-the-fly.js` in the `examples` folder work a working version of this concept.

### Worker Errors

Ponos's worker is designed to retry any error that is not specifically a fatal error. Ponos has been designed to work well with our error library [`error-cat`](https://github.com/Runnable/error-cat).

A fatal error is defined with the `WorkerStopError` class from `error-cat`. If a worker rejects with a `WorkerStopError`, the worker will automatically assume the job can _never_ be completed and will acknowledge the job.

As an example, a `WorkerStopError` can be used to fail a task given an invalid job:

```javascript
const WorkerStopError = require('error-cat/errors/worker-stop-error')
function myWorker (job) {
  return Promise.resolve()
    .then(() => {
      if (!job.host) {
        throw new WorkerStopError('host is required', {}, 'my.queue', job)
      }
    })
    .then(() => {
      return doSomethingWithHost(job)
    })
}
```

This worker will reject the promise with a `WorkerStopError`. Ponos will log the error itself, acknowledge the job to remove it from the queue, and continue with other jobs. You may catch and re-throw the error if you wish to do additional logging or reporting.

Finally, as was mentioned before, Ponos will retry any other errors. `error-cat` provides a `WorkerError` class you may use, or you may throw normal `Error`s. If you do, the worker will catch these and retry according to the server's configuration (retry delay, back-off, max delay, etc.).

```javascript
const WorkerError = require('error-cat/errors/worker-error')
const WorkerStopError = require('error-cat/errors/worker-stop-error')
function myWorker (job) {
  return Promise.resolve()
    .then(() => {
      return externalService.something(job)
    })
    // Note: w/o this catch, the error would simply propagate to the worker and
    // be handled.
    .catch((err) => {
      logErrorToService(err)
      // If the error is 'recoverable' (e.g., network fluke, temporary outage),
      // we want to be able to retry.
      if (err.isRecoverable) {
        throw new Error('hit a momentary issue. try again.')
      } else {
        // maybe we know we can't recover from this
        throw new WorkerStopError(
          'cannot recover. acknowledge and remove job',
          {},
          'this.queue',
          job
        )
      }
    })
}
```

## Worker Options

Currently workers can be defined with a `msTimeout` option. This value defaults to `process.env.WORKER_TIMEOUT || 0`. One can set a specific millisecond timeout for a worker like so:

```js
server.setTask('my-queue', workerFunction, { msTimeout: 1234 })
```

Or one can set this option via `setAllTasks`:

```js
server.setAllTasks({
  // This will use the default timeout...
  'queue-1': queueOneTaskFn,
  // This will use the specified timeout...
  'queue-2': {
    task: queueTwoTaskFn,
    msTimeout: 1337
  }
})
```

## Full Example

```javascript
const ponos = require('ponos')

const tasks = {
  'queue-1': (job) => { return Promise.resolve(job) },
  'queue-2': (job) => { return Promise.resolve(job) }
}

// Create the server
var server = new ponos.Server({
  queues: Object.keys(tasks)
})

// Set tasks for workers handling jobs on each queue
server.setAllTasks(tasks)

// Start the server!
server.start()
  .then(() => { console.log('Server started!') })
  .catch((err) => { console.error('Server failed', err) })

// Or, start using your own hermes client
const hermes = require('runnable-hermes')
const server = new ponos.Server({
  hermes: hermes.hermesSingletonFactory({...})
})

// You can also nicely chain the promises!
server.start()
  .then(() => { /*...*/ })
  .catch((err) => { /*...*/ })
```

## License

MIT

[travis]: https://img.shields.io/travis/Runnable/ponos/master.svg?style=flat-square "Build Status"
[coveralls]: https://img.shields.io/coveralls/Runnable/ponos/master.svg?style=flat-square "Coverage Status"
[dependencies]: https://img.shields.io/david/Runnable/ponos.svg?style=flat-square "Dependency Status"
[devdependencies]: https://img.shields.io/david/dev/Runnable/ponos.svg?style=flat-square "Dev Dependency Status"
[documentation]: https://runnable.github.io/ponos "Ponos Documentation"
[codeclimate]: https://img.shields.io/codeclimate/github/Runnable/ponos.svg?style=flat-square "Code Climate"
