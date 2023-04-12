# Interaction ID & Count

## Introduction

Currently, developers can measure the latency of an event using the [Event Timing API](https://wicg.github.io/event-timing/). Developers may query the [duration](https://w3c.github.io/performance-timeline/#dom-performanceentry-duration), which returns the next paint after event handlers have run. Or they may compute the input delay, the delta between [startTime](https://w3c.github.io/performance-timeline/#dom-performanceentry-starttime) (the event's [timeStamp](https://developer.mozilla.org/en-US/docs/Web/API/Event/timeStamp)) and [processingStart](https://wicg.github.io/event-timing/#dom-performanceeventtiming-processingstart), which is a timestamp taken right before event is dispatched. But measuring event latency separately can’t help developers fully understand the user’s pain. When a user interacts with a web page, a user interaction (i.e. click/tap, press a key, drag)  usually triggers a sequence of events. For example, when the user clicks, several events will be dispatched: pointerdown, pointerup, mousedown, mouseup, click, etc. Therefore, we propose to measure the latency of the whole user interaction by aggregating associated events’ latencies. To calculate an interaction-level latency, there should be a way to group events or to tell if two events are fired by the same user interaction. In this document we propose adding a new ID in PerformanceEventTiming named Interaction ID. Each Interaction ID will represent a unique user interaction. And events triggered by an interaction will share the same Interaction ID.

Interaction Count counts the number of interactions happened since page load, based on the number of unique interaction ID generated. This is NOT necessarily equivalent to the number of unique interaction IDs reported by performance observer, since interactions can be filtered out due to [durationThreshold](https://w3c.github.io/event-timing/#dom-performanceobserverinit-durationthreshold).

## Goal

Enable developers to:
* group events by Interaction ID;
* calculate interaction level latency for a user interaction such as maximum event duration, total event duration or other customized metrics;
* get the total number of interactions happened on the page.

The latency of an interaction can be computed from the latencies of the following events:

| Interaction | Events                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------ |
| keyboard    | keydown, keyup, input(IME only), keypress([WIP](https://github.com/w3c/event-timing/issues/132)) |
| click / tap | pointerdown, pointerup, click                                                                    |
| drag        | pointerdown, pointerup, click                                                                    |

The events in the table are the minimal set chosen to capture the longest [duration](https://w3c.github.io/performance-timeline/#dom-performanceentry-duration) among all events triggered by the interaction. This may prove to be confusing for developers, so we're considerring adding IDs to all PerformanceEventTiming entries corresponding to events considered to be caused by the same interaction.

## Non-Goals

The following are not goals of this proposal:

* Cover all types of interactions or events.
* Provide a new metric.
* Group performance information about events automatically.

## API

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

## Usage

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

## Self-Review Questionnaire: Security and Privacy

> 01.  What information might this feature expose to Web sites or other parties,
> and for what purposes is that exposure necessary?
> 
>      This feature exposes `interactionId` of events exposed to Event Timing, and it is necessary to compute interaction-level latency and compute correctly aggregated data. This feature also exposes `interactionCount` to window performance, and it is necessary for computing aggregated metrics like INP.
> 
> 02.  Do features in your specification expose the minimum amount of information
>      necessary to enable their intended uses?
> 
>      We need a way for developers to know when various PerformanceEventTiming entries correspond to the same user interaction. The `interactionId` is the simplest way to achieve this while still preserving important event-level data (such as how long event handlers take to run). This explainer does not specify the way the id would be computed, but it might be a random long number so developers don't mistakenly rely on it to count interactions or infer some ordering between interactions. The random number would not be reused across frames to prevent any leaks.
>
> 03.  How do the features in your specification deal with personal information,
>      personally-identifiable information (PII), or information derived from
>      them?
>      
>      There's not PII involved in this feature.
>      
> 04.  How do the features in your specification deal with sensitive information?
> 
>      There's no sensitive information involved in this feature.
>      
> 05.  Do the features in your specification introduce new state for an origin
>      that persists across browsing sessions?
>      
>      No.
>      
> 06.  Do the features in your specification expose information about the
>      underlying platform to origins?
>      
>      No.
>      
> 07.  Does this specification allow an origin to send data to the underlying
>      platform?
>      No.
>      
> 08.  Do features in this specification enable access to device sensors?
> 
>      No.
>      
> 09.  What data do the features in this specification expose to an origin? Please
>      also document what data is identical to data exposed by other features, in the
>      same or different contexts.
>      
>      It exposes groups of PerformanceEventTiming entries that correspond to the same user interaction and counting of these groups targetting that website. The granularity is frame/Document, as with all other performance entries.
>      
> 10.  Do features in this specification enable new script execution/loading
>      mechanisms?
>      
>      No.
>      
> 11.  Do features in this specification allow an origin to access other devices?
> 
>      No.
>      
> 12.  Do features in this specification allow an origin some measure of control over
>      a user agent's native UI?
>      
>      No.
>      
> 13.  What temporary identifiers do the features in this specification create or
>      expose to the web?
>      
>      The interaction id's and the number of interactions are temporary identifiers on the user interactions that were handled by the site.
>      
> 14.  How does this specification distinguish between behavior in first-party and
>      third-party contexts?
>      
>      Since information is exposed on frame-granularity, no third-party data is exposed (i.e. the iframe data is exposed only to the iframe).
>      
> 15.  How do the features in this specification work in the context of a browser’s
>      Private Browsing or Incognito mode?
>      
>      The feature does not work differently on incognito as it does not rely on user identifier in any way.
>      
> 16.  Does this specification have both "Security Considerations" and "Privacy
>      Considerations" sections?
>      
>      Event Timing has a privacy/security [section](https://wicg.github.io/event-timing/#priv-sec). It will be augmented with considerations on `interactionId` once that is added to the spec.
>      
> 17.  Do features in your specification enable origins to downgrade default
>      security protections?
>      
>      No.
>      
> 18.  What should this questionnaire have asked?
> 
>      Can't think of other relevant question.