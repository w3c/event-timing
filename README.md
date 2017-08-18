# Event Timing Explainer


Web developers currently have little insight into what causes latency when handling events.

This document provides a minimal proposal which provides developers insight into the cases which are most often slow. This proposal does not address latency of input which isn’t blocked on the browser’s main thread.

## Proposal

A [previous proposal](https://docs.google.com/document/d/15YAIJbv5x4hCVQx3cbUNRd9QSkyvS8bzbAMkcznNSjI/edit#) grew too broad in scope. This proposal explains the minimal API required to solve the following key use cases:

1. Measure event handler / default action duration.
2. Correlate input with slow frames.
    * Including input without handlers, such as input triggering hover.
3. Measure impact of event handlers on scroll performance
    * For scrolls blocked by main thread work.

A polyfill roughly implementing part of this API can be found [here](https://github.com/tdresser/input-latency-web-perf-polyfill/tree/gh-pages).

In order to accomplish these goals, we introduce:

```javascript
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
    // The time the last event handler or default action finished executing.
    // startTime if no event handlers or default action executed.
    readonly attribute DOMHighResTimeStamp processingEnd;
    // Zero.
    readonly attribute DOMHighResTimeStamp duration;
    // Whether or not the event was cancelable.
    readonly attribute boolean cancelable;
    // Whether or not the event contributed to a user scroll.
    readonly attribute boolean eventCausedScroll;
    // Whether or not a commit was pending at processingEnd.
    readonly attribute boolean eventHasCommit;
    // If a commit was pending at processingEnd, the time that commit occurred.
    // Otherwise, 0.
    readonly attribute DOMHighResTimeStamp commitTime;
};
```


### Draw time

See outstanding issues on [defining draw time](#defining-draw-time) and [:visited attacks](#visited-attacks).

The `commitTime` field provides information about when we executed step 7.11 of the [event loop processing model](https://www.w3.org/TR/html51/webappapis.html#event-loops-processing-model), if it was executed.

> For each fully active Document in docs, update the rendering or user interface of that Document and its browsing context to reflect the current state.

However, in the future, we’d like to be able to associate this with when pixels actually hit the screen. The [Frame Timing API](http://wicg.github.io/frame-timing/) will eventually surface more accurate display timestamps, and will also include the time of step 7.11. `commitTime` will enable us to correlate event processing entries with frame timing entries, letting us measure from input time until pixels hit the screen.

### When should we dispatch PerformanceEventTiming entries?

Let’s dispatch `PerformanceEventTiming` for all events for which:

`max(processingEnd, commitTime) - startTime > 50ms`

This considers an event to start when it's timestamp indicates, and an event to end when it’s associated frame is committed, if one exists, and when the event handlers and default action are complete if no associated frame exists.

### What about input triggering multiple DOM event types?

Let's report one entry per DOM event.
* For events which have no listeners, we report one entry per DOM event which would have been dispatched had there been listeners.
* For nested elements with, for example, touchmove event handlers, only one entry is reported, despite there being many listeners.
* For input which triggers multiple DOM events, such as a touch pointer release triggering touchend, pointerend and click, many entries may be dispatched for a single user input.

### Is a Polyfill Good Enough?

Polyfilling this runs into a few problems.

1. Measure event handler duration.
    * It’s possible to polyfill this, but it’s tricky to all combine timing data associated with a single DOM event. See a possible solution [here](https://github.com/tdresser/input-latency-web-perf-polyfill/blob/gh-pages/event_timing.js). This solution doesn’t have adequate performance, as it requires replacing addEventListener, and measuring the time at the end of every event listener invocation.

2. Correlate input with slow frames.
    * This is possible to polyfill for input with event handlers, but impossible for input without event handlers. Adding event handlers for all input types has unacceptable performance overhead.

3. Measure impact of event handlers on scroll performance
    * We can get close to polyfilling this (with the above caveats), by identifying cancellable events of types which could potentially trigger scroll, which executed during a frame in which a scroll occurred. We could remove causedScroll from PerformanceEventTiming and require users to correlate this with scroll events, but including it makes this a fair bit easier for consumers of this API, and will result in slightly higher quality results.


## The Long Tasks API

The long tasks API provides some of the value of this API, but isn’t adequate. The work done in response to an event is often split between multiple tasks. First, the event handlers are executed, which often dirties layout and style. Then when we go to produce a frame, rAF is executed, potentially based on input handled previously, and we perform style and layout. That style and layout needs to be included as part of the cost of event handling, and this isn’t doable using the long tasks API. In order to approximate the user perceived latency of an event, we need to include the time taken to draw the frame, and the long tasks API doesn’t let us do that.

Even if the long tasks API did give us information about any events which triggered this long task, it still wouldn’t solve the problem of correlating all work associated with an event, and wouldn’t threshold on the actual event latency. The event timing API needs to dispatch a performance entry if the total time spent processing the event on the main thread exceeds some threshold.

This API also provides additional context on the event in question.

## Asynchronous Scrolling

Asynchronous scrolling should be essentially equivalent to other asynchronous animations, and will eventually be addressed by extensions to the Frame Timing API. The changes a developer can make to fix high compositing overhead are equivalent for scrolling and other composited animations. Insight into the events which caused scrolling isn’t valuable in this context.

## Cases where we only want to monitor one event type

We may want to support some way of filtering which DOM event types are monitoring at Performance Observer registration time. This could be achieved by extending PerformanceObserverInit. This is beyond the scope of this proposal however, as we should consider providing a general mechanism for this.

## Cases where $THRESHOLD ms is too much Latency to ignore

We could eventually enable configuring the long event threshold, or we could use some heuristics in picking when to report long events, based on the context. For V1, let's stick to using a 50ms threshold.

## Cases where we want more or different timestamps

There are many more useful timestamps, which should be added to these performance entries in the future. Most of this proposal can be incrementally extended by adding additional timestamps to the event or long frame performance entries. The hardest thing to iterate on is the duration that we threshold on for deciding whether or not to fire a long event entry.

The current proposal suggests using main thread frame time (or event handling time, if no frame was produced). There are reasonable arguments to be made that we should instead threshold on our best guess of how long the frame took to produce after input, all the way up until the frame hits the glass. In order to minimize spec churn, I think it makes sense to go with the main thread time for now, and we can consider ways of dealing with cases which fail because we’re using the main thread frame time in the future. If we tune the threshold correctly, as long as we add the estimated glass time to the Frame Timing performance entry, thresholding on the main thread time will be fine.

A few additional pieces of information we should add in the future:

* Event Timing
    * Long tasks V2 style attribution for event handlers.
* Frame Timing
    * Estimated Glass Time
    * rAF handler start/end.
    * Time spent compositing


## Usage
```javascript
// Log performance entries for events which blocked scrolling.

const performanceObserver = new PerformanceObserver((entries) => {
  for (const entry of entries.getEntries()) {
    if (entry.causedScroll && entry.cancellable)
      console.log(entry);
  }
});

performanceObserver.observe({entryTypes:['event']});
```

## Outstanding Issues

### Defining Draw Time
We currently define draw time based on step 7.11 of the [event loop processing model](https://www.w3.org/TR/html51/webappapis.html#event-loops-processing-model), but this needs additional formalization.

### :visited attacks

This proposal opens up a new approach for sniffing a user’s history via :visited. To accomplish this, an attacker would:

* Add a link to $SITE
* Give the link a :visited:hover style which paints
* Avoid giving the link a :hover style which paints
* Give the link element a long (>50ms) mousemove event handler.
* Add an event timing performance observer.

When a performance entry for this mouse move event comes in, then if there’s an associated commit, then the user has previously visited $SITE.

One way to avoid this issue would be to always commit when hovering an object which has a :hover:visited style. Another option would be to make up a timestamp for when we predict we would have executed step 7.11 of the event loop processing model, had it been executed.
