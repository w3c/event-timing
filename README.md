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
    // The time the first event handler started to execute.
    // startTime if no event handlers executed.
    readonly attribute DOMHighResTimeStamp processingStart;
    // The time the last event handler finished executing.
    // startTime if no event handlers executed.
    readonly attribute DOMHighResTimeStamp processingEnd;    
    // The duration between |startTime| and the next execution of step 7.12 in the HTML event loop processing model.
    readonly attribute DOMHighResTimeStamp duration;
    // Whether or not the event was cancelable.
    readonly attribute boolean cancelable;
};
```

Make the following modifications to the "[to dispatch an event algorithm](https://www.w3.org/TR/dom/#dispatching-events)".

Let pendingEntries be an initially empty list of PerformanceEventTiming objects.

Before step one, run these steps:

1.  Let newEntry be a new PerformanceEventTiming object.
1.  Set newEntry's name attribute to event.type.
1.  Set newEntry's entryType attribute to "event".
1.  Set newEntry's startTime attribute to event.timeStamp.
1.  Set newEntry's processingStart attribute to the value returned by performance.now().
1.  Set newEntry's duration attribute to 0.
1.  Set newEntry's cancelable attribute to event.cancelable.

After step 13
* Set newEntry.processingEnd to the value returned by performance.now().
* Append newEntry to pendingEntries.

During step 7.12 of the [event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
* For each fully active `Document` in `docs`, update the rendering or user interface of that `Document` and its browsing context to reflect the current state, and, while doing so, for each `newEntry` in `pendingEntries`:
 * Set newEntry's duration attribute to the value returned by `performance.now() - newEntry.startTime`.
 * If `newEntry.duration > 50`, queue `newEntry`.

### Open Questions

#### Should this apply to all events, or only UIEvents? Perhaps only a subset of UIEvents?

### Usage
```javascript
const performanceObserver = new PerformanceObserver((entries) => {
  for (const entry of entries.getEntries()) {
      console.log(entry);
  }
});

performanceObserver.observe({entryTypes:['event']});
```
