# Custom Performance Entries Explainer

[PerformanceObserver](https://w3c.github.io/performance-timeline/#the-performanceobserver-interface) provides a consistent way for the browser to expose performance information to web developers and analytics packages in the wild. However, there are times when web developers have performance insights they’d like to process alongside the information the browser provides. Web developers should be able to communicate their own information to performance observers, with rich annotation, enabling their centralized performance monitoring logic and third party analytics tooling to integrate data from the browser with insights from the web page.

## Mark and Measure

`performance.mark` and `performance.measure` enable adding some custom information to the performance timeline, but don’t allow associating rich annotations with these entries. These annotations are required for developers to understand the context of performance critical events. Mark and measure also don’t allow using arbitrary timestamps, such as timestamps obtained from events, other performance entries, etc.

## API

A polyfill of the proposed API can be found [here](https://github.com/tdresser/custom-performance-entry).

We propose adding a new method performance.queueEntry, which queues a CustomPerformanceEntry with entryType "custom". The `name`, `startTime`, `duration` and `detail` fields are used to populate the `CustomPerformanceEntry`. `detail` is an additional field on `CustomPerformanceEntry`, which is an unstructured object, analogous to the `detail` field of [`CustomEvent`](https://dom.spec.whatwg.org/#interface-customevent).

```javascript
interface CustomPerformanceEntry : PerformanceEntry {
    readonly attribute any detail;
};

dictionary PerformanceEntryInit {
    DOMString name,
    DOMHighResTimeStamp startTime,
    DOMHighResTimeStamp duration,
    any detail
}

partial interface Performance {
    void queueEntry(PerformanceEntryInit performanceEntryInit);
};

```


## Buffering

These events will be buffered until the end of onload, up to 100 events of entryType "custom". This is the default for entryTypes in general.

## Explicit vs Implicit timestamps

The proposed API always requires passing timestamps for the current time and duration, instead of using the current time, like mark & measure. In some cases, this increases the complexity of the API a bit.

However, there are cases where passing timestamps explicitly is required. For example, consider the case of measuring the time from navigationStart until the page is "interactive", our “[Time to Consistently Interactive](https://developers.google.com/web/tools/lighthouse/audits/consistently-interactive)” metric. In this case, we need to measure a series of timestamps, which are candidates for TTI, but only actually dispatch one of them, several seconds after the fact. This requires explicit timestamps.

## Usage Examples

This example records the time between first contentful paint, and the first touch input. It includes the id of the element touched.

```javascript
let firstTouchStartTime = null;
let firstTouchStartTargetId = null;
let firstContentfulPaintTime = null;

function dispatchPaintToInputIfAvailable() {
  if (firstTouchStartTime == null || firstContentfulPaintTime == null)
    return;
  performance.queueEntry({
    "first-contentful-paint-to-touch-start",
    startTime: firstContentfulPaintTime,
    duration: firstTouchStartTime - firstContentfulPaintTime,
    detail: {id: firstTouchStartTargetId},
  });
}

document.addEventListener('touchstart', (e) => {
  firstTouchStartTime = e.timeStamp;
  firstTouchStartTargetId = e.target ? e.target.id : null;
  dispatchPaintToInputIfAvailable();
}, {once:true, passive:true})

const observer = new PerformanceObserver((entries) => {
  for (const entry of entries.getEntries()) {
    if (entry.name != "first-contentful-paint")
      continue;
    firstContentfulPaintTime = entry.startTime;
    dispatchPaintToInputIfAvailable();
  }
});

observer.observe({entryTypes:['paint']})
```

### Implementing mark & measure with this API
`performance.mark` and `performance.measure` can be redefined in terms of this API.

```javascript
const lastMarkTimes = new Map();

performance.mark = function(name) {
  const now = performance.now();
  lastMarkTimes.set(name, now);
  performance.queueEntry({
    name,
    startTime: now,
    duration: 0,
  });
}

performance.measure = function(name, startmark, endmark) {
  performance.queueEntry({
    name,
    startTime: lastMarkTimes.get(startMark),
    duration: lastMarkTimes.get(endMark) - lastMarkTimes.get(startMark),
  });
}
```
