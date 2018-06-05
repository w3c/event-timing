# Event Timing Web Perf API

Monitoring event latency today requires an event listener. This precludes measuring event latency early in page load, and adds unnecessary performance overhead.

This document provides a proposal for giving developers insight into the latency of a subset of events triggered by user interaction.

## Minimal Proposal

We propose exposing performance information for events of the following types when they take longer than 50ms from timestamp to the next paint.
* MouseEvents
* PointerEvents
* TouchEvents
* KeyboardEvents
* WheelEvents
* InputEvents
* CompositionEvents

This proposal defines an API addressing the following use cases:

1.  Observe the queueing delay of input events before event handlers are registered.
2.  Measure combined event handler duration, including browser event handling logic.

See more specific use cases [here](#specific-use-cases).

A polyfill approximately implementing this API can be found [here](https://github.com/tdresser/input-latency-web-perf-polyfill/tree/gh-pages).

Only knowing about slow events doesn't provide enough context to determine if a site is getting better or worse. If a site change results in more engaged users, and the fraction of slow events remains constant, we expect an increase in the number of slow events. We also need to enable computing the fraction of events which are slow to avoid conflating changes in event frequency with changes in event latency.

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
    // |startTime| if no event handlers executed.
    readonly attribute DOMHighResTimeStamp processingStart;
    // The time the last event handler finished executing.
    // |startTime| if no event handlers executed.
    readonly attribute DOMHighResTimeStamp processingEnd;    
    // The duration between |startTime| and the next execution of step 7.12 in the HTML event loop processing model.
    readonly attribute DOMHighResTimeStamp duration;
    // Whether or not the event was cancelable.
    readonly attribute boolean cancelable;
};

// Contains the number of events which have been dispatched, per event type.
interface EventCounts {
  readonly attribute unsigned long click;
  ...
  readonly attribute unsigned long touchmove;
  ...
};

partial interface Performance {
    // Contains the number of events which have been dispatched, per event type. Populated asynchronously. 
    readonly attribute EventCounts eventCounts;
};
```

Make the following modifications to the "[to dispatch an event algorithm](https://www.w3.org/TR/dom/#dispatching-events)".

Let `pendingEventEntries` be an initially empty list of `PerformanceEventTiming` objects, stored per document.

Before step one, run these steps:

1. If `event` is none of: "MouseEvent", "PointerEvent", "TouchEvent", "KeyboardEvent", "WheelEvent", "InputEvent", "CompositionEvent" or `event.isTrusted` is false, then return.
1. Let newEntry be a new `PerformanceEventTiming` object.
1. Set newEntry's name attribute to `event.type`.
1. Set newEntry's entryType attribute to "event".
1. Set newEntry's startTime attribute to `event.timeStamp`.
1. If `event.type` is "pointermove", set newEntry's startTime to `event.getCoalescedEvents()[0].startTime`.
1. Set newEntry's processingStart attribute to the value returned by `performance.now()`.
1. Set newEntry's duration attribute to 0.
1. Set newEntry's cancelable attribute to `event.cancelable`.

After step 13
* Set `newEntry.processingEnd` to the value returned by `performance.now()`.
* Append `newEntry` to `pendingEventEntries`.

After step 7.12 of the [event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
* For each `newEntry` in `pendingEventEntries`:
  * Set newEntry's duration attribute to the value returned by 
    * ```Math.ceil((performance.now() - newEntry.startTime)/8) * 8```
    * This value is rounded up to the next 8ms to avoid providing a high resolution timer.
  * Increment `performance.eventCounts[newEntry.name]`.
  * If `newEntry.duration > 50 && newEntry.processingStart != newEntry.processingEnd`, queue `newEntry` on the current document.
  * If `newEntry.duration > 50 && newEntry.processingStart == newEntry.processingEnd`, the user agent MAY queue `newEntry` on the current document.

In the case where event handlers took no time, a user agent may opt not to queue the entry. This provides browsers the flexibility to ignore input which never blocks on the main thread.

### Security and Privacy
To avoid adding another high resolution timer to the platform, `duration` is rounded to the nearest multiple of 8. Event handler duration inherits it's precision from `performance.now()`, and could previously be measured by overriding addEventListener, as demonstrated in the polyfill.

### Usage
```javascript
const performanceObserver = new PerformanceObserver((entries) => {
  for (const entry of entries.getEntries()) {
      console.log(entry);
  }
});

performanceObserver.observe({entryTypes:['event']});
```

## First Input Timing
The very first user interaction has a disproportionate impact on user experience, and is often disproportionately slow. In Chrome, the 99'th percentile of the `entry.processingStart` - `entry.startTime` for the following events is over 1 second:
* Key down
* Mouse down
* Pointer down which is followed by a pointer up
* Click

This is ~4x the 99'th percentile of these events overall. In the median, we see ~10ms for the first event, and ~2.5ms for subsequent events.

This list intentionally excludes scrolls, which are often not blocked on javascript execution.

In order to address capture user pain caused by slow initial interactions, we propose a small addition to the event timing API specific to this use-case.

Let `pendingPointerDown` be `null`.
Let `hasDispatchedEvent` be set to `false` on navigationStart.

When iterating through the entries in `pendingEventEntries`, after dispatching `newEntry`:
  * If `hasDispatchedEvent` is `true`, return.
  * Let `newFirstInputDelayEntry` be a copy of `newEntry`.
  * Set `newFirstInputDelayEntry.entryType` to `firstInput`.
  * If `newFirstInputDelayEntry.type` is "pointerdown"`:
    * Set `pendingPointerDown` to newFirstInputDelayEntry
    * return
  * Set `hasDispatchedEvent` to `true`.
  * If `newFirstInputDelayEntry.type` is "pointerup":
    * Queue `pendingPointerDown`
  * If `newFirstInputDelayEntry.type` is one of "click", "keydown" or "mousedown":
    * Queue `newFirstInputDelayEntry`
      
FirstInputDelay can be polyfilled today: see [here](https://github.com/GoogleChromeLabs/first-input-delay) for an example. However, this requires registering analytics JS before any events are processed, which is often not possible. First Input Delay can also be polyfilled on top of the event timing API, but it isn't very ergonomic, and due to the asynchrony of `performance.eventCounts` can sometimes incorrectly report an event as the first event when there was a prior event less than 50ms.

## Specific Use Cases
* Clicking a button changes the sorting order on a table. Measure how long it takes from the click until we display reordered content.
* A user drags a slider to control volume. Measure the latency to drag the slider. 
 * Note that part of the latency may come from event handlers, but a site may choose to coalesce input during the frame, and respond to it during rAF.
* Hovering a menu item triggers a flyout menu. Measure the latency for the flyout to appear.
* Measure the 75'th percentile of click event queueing times.
