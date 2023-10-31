# Interaction ID & Count

## Background

When a user interacts with a web page (i.e. click, tap, and keyboard) this usually triggers a [sequence of events](https://www.w3.org/TR/uievents/). For example, when the user clicks, several events will be dispatched: pointerdown, pointerup, mousedown, mouseup, click, etc.  Which events are dispatched can even be conditional: hover, focus, blur, change events, etc, all depend on the target of the event and the context of the page.

Although measuring the latency of each individual event using Event Timing API is useful, it takes careful effort to interpret this data in a way that best represents user percieved responsiveness.  Therefore, we want a way to simplify the grouping of separate Event Timings into common interactions so we can better represent perceived interaction latencies.

Additionally, we would like to simplify the task of counting the total number of interactions in a way that matches user intuitions: count the total number of clicks, taps, keypresses -- rather than counting the nunber of UI events which were dispatched, which is not user visible.

## What is an InteractionId?

An `interactionId` is an number which uniquely identifies a user interaction.  Most events will not have an `interactionId` (or have a value of 0), as they are not dispatched due to specific discrete interactions. Event which do have an `interactionId` assigned, can be grouped together with other events sharing the same `interactionId`. This enables us to aggregate event latencies from the same interaction and calculate an overall interaction latency, typically using the single longest event `duration`.

## What is InteractionCount?

InteractionCount counts the number of interactions that have happened since page load, based on the number of unique interaction ID generated. This is NOT necessarily equivalent to the number of unique interaction IDs reported by performance observer, since interactions can be filtered out due to [durationThreshold](https://w3c.github.io/event-timing/#dom-performanceobserverinit-durationthreshold).

## Where are we now?

Only certain event types may expose an `interactionId`; Those events which are required to capture overall interaction latency.  These are listed below:

| Interaction Type | Event Types with InteractionId Exposed                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------ |
| keyboard    | keydown, keyup, input(IME only), keypress([WIP](https://github.com/w3c/event-timing/issues/134)), keydown/up under IME(WIP) |
| click / tap | pointerdown, pointerup, click, contextmenu(WIP)                                                                    |
| drag        | pointerdown, pointerup, click                                                                    |

These events will also not always have an `interactionId` in all cases.  For example, some pointerdown events are followed by pointercancel (scrolling) and are not considered a discrete interaction.  Event Timings which are not Interactions will always get an `interactionId` of `0`.

## API Definition

Interaction ID is an attribute on [PerformanceEventTiming](https://wicg.github.io/event-timing/#sec-performance-event-timing):

```js
interface PerformanceEventTiming : PerformanceEntry {
    // The ID for the user interaction that caused this event.
    readonly attribute unsigned long interactionId;
};
```

Interaction Count is an attribute on [Performance](https://w3c.github.io/event-timing/#sec-extensions).
```js
interface Performance {
    // The number of interactions recorded on this window performance object.
    readonly attribute unsigned long long interactionCount;
};
```

## Usage Example

The following example computes, for each interaction, the maximum event duration for all events corresponding to that interaction:

```js
// Hashmap for storing event latencies. The key is Interaction ID.
var map = new Map();

const performanceObserver = new PerformanceObserver((entries) => {
    for (const entry of entries.getEntries()) {
        if (entry.interactionId > 0) {
            // Include the event 'duration', which captures from hardware timestamp to next paint after handlers run.
            const duration = entry.duration;
            // Include the Interaction ID.
            const interaction_id = entry.interactionId;
            if (!map.has(interaction_id)) {
                map[interaction_id] = [];
            }
            map[interaction_id].push(duration);
        }
    }
});

performanceObserver.observe(
    {
        type: 'event',
        durationThreshold: 16, // Minimum supported by the spec.
        buffered: true
    }
);
```

Some time later, after you've interacted a few times on the page, run the following code example to print interaction durations & count:

```js
// Print the maximum event duration for each user interaction.
Object.entries(map).forEach(([k, v]) => {
    console.log(`interactionId: ${k}, max event duration: ${Math.max(...v)}`);
});

// Print the total number of interactions happened so far.
console.log(`interaction count: ${window.performance.interactionCount}`);
```
