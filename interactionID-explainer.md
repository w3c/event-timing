# Interaction ID & Count

## Background

When a user interacts with a web page, a user interaction (i.e. click/tap, press a key, drag) usually triggers a sequence of events. For example, when the user clicks, several events will be dispatched: pointerdown, pointerup, mousedown, mouseup, click, etc. Measuring the latency of each individual event can't reflect user perceptive responsiveness. Therefore, we want a way to group events up into interactions so we can measure interaction latencies. And further counting the number of interactions on the page so we can aggregate all interactions and calculate the overall page responsiveness of a user visit. For example, calculating INP - the 98th percentile of interaction durations over a page load.

## What is InteractionId?

The interactionId is an Id which uniquely identifies an user interaction. Each event has an interactionId assigned, so by comparing the interactionId of two events, we can tell if the two events are fired by the same user interaction. Thus it enables us to aggregate event latencies from the same interaction and calculate an interaction-level latency.

## What is InteractionCount?

InteractionCount counts the number of interactions that have happened since page load, based on the number of unique interaction ID generated. This is NOT necessarily equivalent to the number of unique interaction IDs reported by performance observer, since interactions can be filtered out due to [durationThreshold](https://w3c.github.io/event-timing/#dom-performanceobserverinit-durationthreshold).

## Where are we now?

InteractionId is currently not exposed to all event types for implementation complexity reasons. Only certain important event types which we think are just enough to capture the interaction latency are exposed and listed as below:

| Interaction Type | Event Types with InteractionId Exposed                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------ |
| keyboard    | keydown, keyup, input(IME only), keypress([WIP](https://github.com/w3c/event-timing/issues/134)), keydown/up under IME(WIP) |
| click / tap | pointerdown, pointerup, click, contextmenu(WIP)                                                                    |
| drag        | pointerdown, pointerup, click                                                                    |

Event types that are not exposed to interactionId will always get an interactionId of 0, which is a trivial and invalid value.

We have current work in progress to include more event types like keypress, keydown/up under IME, contextmenu, etc. Ideally, we would want all event types to have interactionId exposed. However, those that have no impact on the interaction latency calculations are considered not important, thus off our top priorities.

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
var map = {};

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

// Print the maximum event duration for each user interaction.
Object.entries(map).forEach(([k, v]) => {
    console.log(Math.max(...v));
});
```

The following example prints interaction count:

```js
// Print the total number of interactions happened so far.
window.performance.interactionCount;
```
