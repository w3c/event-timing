# Computing Interaction to Next Paint from Event Timing

[Interaction to Next Paint (INP)](https://web.dev/articles/inp) summarizes a page's responsiveness using data from the [Event Timing API](https://w3c.github.io/event-timing/).
Libraries compute the metric from the entries the API exposes.
This explainer describes that calculation for one window and the option of using its longest interaction.
It does not add requirements to the API specification.

## Group events into interactions

One tap can produce several `PerformanceEventTiming` entries.
Group entries by their nonzero [`interactionId`](https://w3c.github.io/event-timing/#dom-performanceeventtiming-interactionid), ignoring entries whose ID is zero.
The interaction's latency is the longest `duration` among its entries, rather than their sum.
For example, entries with durations of 40 ms, 80 ms and 56 ms for the same interaction give a latency of 80 ms.

## Choose a summary value

The longest interaction is a simple summary: sort the interaction latencies longest first and take the first value.
This keeps a single slow interaction visible, even on a page with many interactions.

Google's Core Web Vitals definition of INP discounts one slow interaction for every 50 interactions.
Use [`performance.interactionCount`](https://w3c.github.io/event-timing/#dom-performance-interactioncount) for this count, including interactions whose entries the observer did not receive.
The zero-based index in the sorted list is `Math.floor(performance.interactionCount / 50)`.
With fewer than 50 interactions, both approaches select the longest interaction.
At 120 interactions, the Core Web Vitals calculation selects the third longest.

For example, suppose the four longest latencies are 200 ms, 160 ms, 120 ms and 80 ms, with the remaining interactions all faster:

| Total interactions | Longest interaction | Core Web Vitals INP |
| --- | --- | --- |
| 49 | 200 ms | 200 ms |
| 50 | 200 ms | 160 ms |
| 120 | 200 ms | 120 ms |

[Michal's explanation](https://github.com/w3c/event-timing/issues/157#issuecomment-6686011584) describes the discount as a way to smooth occasional delays on long-lived, highly interactive pages.
Using the longest interaction is also useful when deriving a responsiveness metric from Event Timing.
To compare results with Core Web Vitals, use its discount rather than the longest-interaction alternative.

## Example

This example keeps the maximum duration for each observed interaction and calculates both candidates when new entries arrive.
`undefined` means no interaction latency has been observed yet.

```js
const latencies = new Map();

new PerformanceObserver(list => {
    for (const entry of list.getEntries()) {
        if (entry.interactionId > 0) {
            const latency = latencies.get(entry.interactionId) ?? 0;
            latencies.set(entry.interactionId, Math.max(latency, entry.duration));
        }
    }
    const sorted = [...latencies.values()].sort((a, b) => b - a);
    const longestInteraction = sorted[0];
    const skip = Math.floor(performance.interactionCount / 50);
    const inp = sorted[Math.min(skip, sorted.length - 1)];
}).observe({ type: 'event', buffered: true, durationThreshold: 16 });
```

The candidates can change as the user interacts, so update them before reporting.
This example only illustrates grouping and selection in one window.
Production reporting also needs to handle pending observer entries, the page lifecycle and restored pages.
It does not combine interactions from other frames.

## Observation thresholds

An event observer's default [`durationThreshold`](https://w3c.github.io/event-timing/#dom-performanceobserverinit-durationthreshold) is 104 ms, and the lowest threshold is 16 ms.
Registering with a threshold of 16 ms collects faster events from then on.
`buffered: true` also requests earlier buffered entries, but the event buffer only retains entries of at least 104 ms.
Faster interactions still contribute to `performance.interactionCount` even when their entries are missing.

If the discount goes past the observed list, the example falls back to its last latency.
That fallback is an estimate: entries below the applicable threshold cannot be recovered from the observed list.
An empty list does not establish an INP of zero.

## Keeping fewer interactions

An implementation need not keep every interaction.
Keeping the 10 longest observed interaction latencies gives the same selection as keeping them all while the total interaction count is below 500.
At 500 interactions the discount selects index 10, which is the eleventh longest, so a list of 10 can only provide a fallback.
The longest-interaction alternative only needs the maximum observed latency.

## Further reading

- [INP calculation discussion](https://github.com/w3c/event-timing/issues/157)
- [Core Web Vitals INP definition](https://web.dev/articles/inp)
- [web-vitals INP implementation](https://github.com/GoogleChrome/web-vitals/blob/main/src/onINP.ts)
