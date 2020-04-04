![CI](https://github.com/patrickkunka/bandwidth-throttle-stream/workflows/CI/badge.svg) [![Coverage Status](https://coveralls.io/repos/github/patrickkunka/bandwidth-throttle-stream/badge.svg?branch=master)](https://coveralls.io/github/patrickkunka/bandwidth-throttle-stream?branch=master)

# Bandwidth Throttle Stream

A Node.js transform stream for throttling bandwidth which distributes available bandwidth evenly between all requests in a "group", accurately simulating the effect of network conditions on simultaneous overlapping requests.

#### Features
- Idiomatic pipeable Node.js transform stream API
- Distributes the desired bandwidth evenly over each second
- Distributes the desired bandwidth evenly between all active requests
- Abortable requests ensure bandwidth is redistributed if a client aborts a request

#### Contents
- [Usage](#usage)
    - [Creating a Group](#creating-a-group)
    - [Attaching Throttles](#attaching-throttles)
    - [Handling Output](#handling-output)
- [Configuration Options](#configuration-options)
- [Dynamic Configuration](#dynamic-configuration)
- [Aborting Requests](#aborting-requests)
- [Destroying Requests](#destroying-requests)

## Usage

#### Creating a Group

We must firstly create a "bandwidth throttle group" which will be configured with a specific throughput in bytes (B) per second.

```js
import {createBandwidthThrottleGroup} from 'bandwidth-throttle-stream';

// Create a group with a configured available bandwidth in bytes (B) per second.

const bandwidthThrottleGroup = createBandwidthThrottleGroup({
    bytesPerSecond: 500000 // 500KB/s
});
```

Typically we would only create a single group for a server running a simulation, which all requests to be throttled are routed through, but could also create multiple groups if we wanted to run multiple simulations with different configurations through the same server.

#### Attaching Throttles

Once we've created a group, we can then attach individual pipeable "throttles" to it, as requests come into our server.

Typically, we would insert a throttle in between a readable stream (e.g file system readout, HTTP response), and the response stream of the incoming request to be throttled.

```js
// Attach a throttle to a group (e.g. in response to an incoming request)

const throttle = bandwidthThrottleGroup.createBandwidthThrottle();

// Throttle the response by piping a readable stream to a writable
// stream via the throttle

someReadableStream
    .pipe(throttle)
    .pipe(someWritableStream);

```

#### Handling Output

In most cases, we need more granular control of data output than simply piping to a writable stream (for example when throttling an HTTP request).

In these cases, we can use any of the Node.js stream events available such as `data` and `end`:

```js
someReadableStream
    .pipe(throttle)
    .on('data', chunk => response.write(chunk)
    .on('end', () => {
        // Set the response status of the HTTP request to 200
        response.status(200);
        // End the request
        response.end();
        // Destroy the throttle to release it from the group
        throttle.destroy();
    });
```

## Configuration Options

Each bandwidth throttle group accepts an optional object of configuration options:

```js
const bandwidthThrottleGroup = createBandwidthThrottleGroup({
    bytesPerSecond: 500000 // 500KB/s,
    resolutionHz: 20 // aim to write output 20x per second
});
```

The following options are available.

```ts
interface IConfig {
    /**
     * The maximum number of bytes allowed to pass through the
     * throttle, each second.
     *
     * @default Infinity
     */

    bytesPerSecond?: number;

    /**
     * Defines how frequently the processing of bytes should be
     * distributed across each second.
     *
     * A higher resolution will ensure smoother throttling, but
     * will be more expensive computationally and will be constrained
     * by the performance of the javascript runtime.
     *
     * @default 40
     */

    resolutionHz?: number;
}

export default IConfig;
```

## Dynamic Configuration

A group can be reconfigured at any point after creation via its `.configure()` method, which accepts the same configuration interface as the `createBandwidthThrottleGroup()` factory.

```js
// Create a group with no throttling

const bandwidthThrottleGroup = createBandwidthThrottleGroup();

// ... after some configuration event:

bandwidthThrottleGroup.configure({
    bytesPerSecond: 6000000
})
```

## Aborting Requests

When a client aborts a requests, its important that we also abort the throttle, ensuring the group can re-balance available bandwidth correctly.

```js
const throttle = bandwidthThrottleGroup.createBandwidthThrottle();

request.on('aborted', () => {
    // Client aborted request

    throttle.abort();
});

someReadableStream
    .pipe(throttle)
    .on('data', chunk => response.write(chunk)
    .on('end', () => {
        // Set the response status of the HTTP request to 200
        response.status(200);
        // End the request
        response.end();
        // Destroy the throttle to release it from the group
        throttle.destroy();
    });
```

## Destroying Requests

To prevent memory leaks, individual throttles should be destroyed once all data for a request has been processed, and the request as ended.

This ensures the throttle instance is fully released from its parent group.

Each throttle instance exposes a `.destroy()` method for this purpose:

```js
throttle.destroy();
```