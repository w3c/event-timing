# Event Timing Web Perf API

Monitoring event latency today requires an event listener. This precludes measuring event latency early in page load, and adds unnecessary performance overhead.

This document provides a proposal for giving developers insight into all event latencies.


## Minimal Proposal

This proposal explains the minimal API required to solve the following use cases:

1.  Observe the queueing time of input events.
    *   This enables us to measure [First Input Delay](https://docs.google.com/document/d/1Tnobrn4I8ObzreIztfah_BYnDkbx3_ZfJV5gj2nrYnY/edit).
2.  Measure combined event handler duration.

A polyfill implementing this API can be found [here](https://github.com/tdresser/input-latency-web-perf-polyfill/tree/gh-pages).

In order to accomplish these goals, we introduce:


```
interface PerformanceEventTiming : PerformanceEntry {
    // The type of event dispatched. E.g. "touchmove".
    // Doesn't require an event listener of this type to be registered.
    readonly attribute DOMString name;
    // "event".
    readonly attribute DOMString entryType;
    // The event timestamp.
    readonly attribute DOMHighResTimeStamp startTime;
    // The time the first event handler or default action started to execute.
    // startTime if no event handlers or default action executed.
    readonly attribute DOMHighResTimeStamp processingStart;
    // The duration between when the last event handler or default action finished executing
    // and |startTime|.
    // 0 if no event handlers or default action executed.
    readonly attribute DOMHighResTimeStamp duration;
    // Whether or not the event was cancelable.
    readonly attribute boolean cancelable;
};
```


When the **performance event timing entry dispatch algorithm** is invoked with a PerformanceEventTiming object |newEntry|, and a boolean |executedListeners|, execute the following steps:

1.  If executedListeners is false, return.
2.  If newEntry.duration < 50 and the performance entry buffer contains an entry with entryType |newEntry.entryType| return.
3.  Queue newEntry.
4.  Add newEntry to the performance entry buffer.

Make the following modifications to the "[to dispatch an event algorithm](https://www.w3.org/TR/dom/#dispatching-events)".

Before step one, run these steps:



1.  Let executedListeners be false
2.  Let newEntry be a new PerformanceEventTiming object
3.  Set newEntry's name attribute to event.type.
4.  Set newEntry's entryType attribute to "event".
5.  Set newEntry's startTime attribute to event.timeStamp.
6.  Set newEntry's processingStart attribute to the value returned by performance.now().
7.  Set newEntry's duration attribute to 0.
8.  Set newEntry's cancelable attribute to event.cancelable.

After step 6
*   if any event listeners or a default action were executed, set executedListeners to true.

After step 13
*   set newEntry.duration to the value returned by performance.now() - event.timeStamp
*   execute the **performance event timing entry dispatch algorithm **on newEntry and executedListeners.


### Open Questions

#### Should this apply to all events, or only UIEvents?

#### How should we handle cases where the default action doesn't block javascript?

For example composited scrolling? I suspect behaving as though the duration is 0 is correct, but specifying this may prove tricky.


### Usage
```javascript
// Log performance entries for events which blocked scrolling.

const performanceObserver = new PerformanceObserver((entries) => {
  for (const entry of entries.getEntries()) {
      console.log(entry);
  }
});

performanceObserver.observe({entryTypes:['event']});
```
