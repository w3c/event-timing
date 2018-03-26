# Event Timing Web Perf API

Monitoring event latency today requires an event listener. This precludes measuring event latency early in page load, and adds unnecessary performance overhead.

This document provides a proposal for giving developers insight into all event latencies.

## Minimal Proposal

This proposal explains the minimal API required to solve the following use cases:

1.  Observe the queueing delay of input events before event handlers are registered.
2.  Measure combined event handler duration.

A polyfill approximately implementing this API can be found [here](https://github.com/tdresser/input-latency-web-perf-polyfill/tree/gh-pages).

To accomplish these goals, we introduce:

```js
interface PerformanceEventTiming : PerformanceEntry {
    // The type of event dispatched. E.g. "touchmove".
    // Doesn't require an event listener of this type to be registered.
    readonly attribute DOMString name;
    // "event".
    readonly attribute DOMString entryType;
    // The event timestamp.
    readonly attribute DOMHighResTimeStamp startTime;
    // The start time of the operation during which the event was dispatched.
    readonly attribute DOMHighResTimeStamp processingStart;
    // The duration between when the operation during which the event was
    // dispatched finished executing and |startTime|.
    readonly attribute DOMHighResTimeStamp duration;
    // Whether or not the event was cancelable.
    readonly attribute boolean cancelable;
};
```

When beginning an operation which will dispatch an event `event`, execute these steps:
 1.  Let `newEntry` be a new `PerformanceEventTiming` object.
 1.  Set `newEntry`'s `name` attribute to `event.type`.
 1.  Set `newEntry`'s `entryType` attribute to "event".
 1.  Set `newEntry`'s `startTime` attribute to `event.timeStamp`.
 1.  Set `newEntry`'s `processingStart` attribute to the value returned by `performance.now()`.
 1.  Set `newEntry`'s `duration` attribute to 0.
 1.  Set `newEntry`'s `cancelable` attribute to `event.cancelable`.

After the operation during which `event` was dispatched, execute these steps:
 1.  Set `newEntry.duration` to the value returned by `performance.now() - event.timeStamp`.
 1.  If `event.isTrusted` is true and `newEntry.duration` > 50:
  1.   Queue `newEntry`.
  1.   Add `newEntry` to the performance entry buffer.

### Open Questions

#### Should this apply to all events, or only UIEvents?

#### How should we handle cases where the operation during which the event was dispatched doesn't block javascript?

For example composited scrolling? I suspect behaving as though the duration is 0 is correct, but specifying this may prove tricky.


### Usage
```javascript
const performanceObserver = new PerformanceObserver((entries) => {
  for (const entry of entries.getEntries()) {
      console.log(entry);
  }
});

performanceObserver.observe({entryTypes:['event']});
```
