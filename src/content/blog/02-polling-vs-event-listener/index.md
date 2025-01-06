---
title: "Polling vs Event Listener"
description: "Exploring the difference between polling and listening to events"
date: "Jan 04 2025"
draft: false
---
Let's say we have a program that has to wait for a resource to be completely configured or loaded. In this situation we must find a way to check its configuration or setup state. 

One way of doing it is through polling - checking its state at intervals of time-.

Node allows us to schedule procedures in intervals with the `setInterval` method of the `TimeOut` class. We can record current minute and second in a given interval in `ms`:

```ts
setInterval(() => {
    const date = new Date();
    console.log(`${date.getMinutes()}:${date.getSeconds()}`);
}, 1000); 
```

This procedure can be wrapped in a `Promise` that checks a `loaded` flag to determine its resolution.

```ts
let loaded = false;

function isLoaded(): Promise<void> {
  return new Promise((resolve) => {
    const _interval = setInterval(() => {
      if (!loaded) return;
      resolve();
      clearInterval(_interval);
    });
  });
}
```

To simulate a resource being loaded or configured, we'll call `setTimeOut` to change `loaded` to `true` after one second. Additionally, we'll pass `setInterval` a `delay` of `10ms` and add a counter to keep track of how many times the`loaded` flag was checked before `Promise` was resolved.
 
```ts
let loaded = false;
let counter = 0;

main();

async function main() {
  console.log("INIT - loaded state: ", loaded);
  startLoading();
  await isLoaded();
  console.log("Loaded!");
}
  
function startLoading() {
  setTimeout(() => {
    loaded = true;
  }, 1000);
}

function isLoaded(): Promise<void> {
  return new Promise((resolve) => {
    const _interval = setInterval(() => {
      console.log(`Pooling - count: ${counter} - load state: `, loaded);
      counter++;
      if (!loaded) return;
      resolve();
      clearInterval(_interval);
    }, 10);
  });
}
```

Running the code above shows that the `loaded` flag was checked about 60 times in one second. It could be twice as much with a delay of 1ms. 

Despite being a simple variable check, we could run into problems if the procedure inside our `Promise` was more resource-intensive. We could be running the risk of blocking the event loop unnecessarily sending several requests to a server.

>According to the Node docs, the default `setInterval` delay is 1ms https://nodejs.org/api/timers.html#settimeoutcallback-delay-args. Imagine many clients polling a web server at intervals of 1ms per second. This would be analogous to a “distributed denial of service” (DDoS) attack.

In a more resourceful approach, we would take advantage of Node's event-driven architecture. By emitting an event when the resource is loaded and defining a related listener within `Promise`.

```ts
import EventEmitter from "node:events";

const eventEmitter = new EventEmitter();

main();

async function main() {
    console.log("INIT");
    startLoading();
    await isLoaded();
    console.log("END");
}

function startLoading() {
    setTimeout(() => {
        eventEmitter.emit("loaded");
    }, 1000);
}

function isLoaded(): Promise<void> {
    return new Promise((resolve) => {
        console.log("Check loaded");
        eventEmitter.once("loaded", resolve);
    });
}
```

In this way, no wasteful checks would be made.

Now, a catchup, let's imagine we've got a very fast connection (and not an arbitrary `setInterval`) and our resource finishes loading before we await for `isLoaded()`. We would set `"loaded"` event listener in our `startLoading()` Promise AFTER emitting `"loaded"` when resource is finished loading, thus never resolving the promise.

We could handle that edge-case by using a `isLoaded` variable to store the loading state. When calling `isLoaded()` we first check if variable is true, resolving the Promise. If false, we set `"loaded"` listener to resolve it as soon as the event is emitted.  

```ts
import EventEmitter from "node:events";

const eventEmitter = new EventEmitter();

main();

let isLoaded = false;

async function main() {
    console.log("INIT");
    startLoading();
    await isLoaded();
    console.log("END");
}

function startLoading() {
    setTimeout(() => {
        eventEmitter.emit("loaded");
    }, 1000);
}

function isLoaded(): Promise<void> {
    return new Promise((resolve) => {
        console.log("Check loaded");
        if (isLoaded) return resolve();
        eventEmitter.once("loaded", resolve);
    });
}
```

Now we've got a more resourceful approach, taking advantage of Node's event-driven architecture, avoiding hundreds of unnecessary calls made by `setInterval()`.