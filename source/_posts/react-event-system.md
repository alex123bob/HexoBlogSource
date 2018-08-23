---
title: React Event System
date: 2018-08-23 10:19:50
tags:
---

I just do a few anatomy of react source code on its event system, and this is not the latest source code coz Facebook has already updated React into a new one featured with Fiber which will optimize React's reaction towards user's input and output while React Component is under its own life cycle process.

````Javascript
render() {
    return (
        <div onClick={(evt => {
            console.log(JSON.stringify(evt))
        })}>
            All Starts With A Big Bang
        </div>
    )
}
````

> **_UpdateDOMProperties** [At the very first]

In file `ReactDOMComponent.js`, `_UpdateDOMProperties` will be triggered when mountComponent and updateCompoennt. It's just like appending properties or updating properties during its life cycle.
Let's jump to the point, `enqueuePutListener`. This is responsible for registering event.

> **So go to definition of `enqueuePutListener`**

````Javascript
function enqueuePutListener(inst, registrationName, listener, transaction) {
  if (transaction instanceof ReactServerRenderingTransaction) {
    return;
  }
  if (__DEV__) {
    // IE8 has no API for event capturing and the `onScroll` event doesn't
    // bubble.
    warning(
      registrationName !== 'onScroll' || isEventSupported('scroll', true),
      "This browser doesn't support the `onScroll` event",
    );
  }
  var containerInfo = inst._hostContainerInfo;
  var isDocumentFragment =
    containerInfo._node && containerInfo._node.nodeType === DOC_FRAGMENT_TYPE;
  var doc = isDocumentFragment
    ? containerInfo._node
    : containerInfo._ownerDocument;
  listenTo(registrationName, doc);
  transaction.getReactMountReady().enqueue(putListener, {
    inst: inst,
    registrationName: registrationName,
    listener: listener,
  });
}
````
Too much info, but what we need know is: 
1. This functionality traces back to instance's ancestor until we find document. That's why React Event is bound to document. 
2. Then invoking `listenTo` will register event to document. 
3. In the end, transaction will call `putListener` to store this event, when event is triggered, React will call this event's callback func.

> **Let's dive into `listenTo`**

It's defiend in file `ReactBrowserEventEmitter.js`. Within `listenTo`, the core is `trapBubbledEvent` and `trapCapturedEvent`. Bubbled is for event at the phase of bubbling, captured is for capturing. Here we just take a look at `trackBubbledEvent`.

````Javascript
trapBubbledEvent: function(topLevelType, handlerBaseName, element) {
    if (!element) {
      return null;
    }
    return EventListener.listen(
      element,
      handlerBaseName,
      ReactEventListener.dispatchEvent.bind(null, topLevelType),
    );
}
````

As you can see in code snippet above, `EventListener` does only one thing, `listen`. And if you go deeper to see what exactly EventListener.listen does, you will finally see something you're familiar with. (addEventListener/attachEvent)

````Javascript
listen: function listen(target, eventType, callback) {
    if (target.addEventListener) {
      target.addEventListener(eventType, callback, false);
      return {
        remove: function remove() {
          target.removeEventListener(eventType, callback, false);
        }
      };
    } else if (target.attachEvent) {
      target.attachEvent('on' + eventType, callback);
      return {
        remove: function remove() {
          target.detachEvent('on' + eventType, callback);
        }
      };
    }
}
````

> **Time to store event**

In aforementioned section, we have already `listened` event, but where and how do we process event's listener(AKA event callback). Here is the answer. You can find `putListener` in functionality `enqueuePutListener`.

````Javascript
function putListener() {
  var listenerToPut = this;
  EventPluginHub.putListener(
    listenerToPut.inst,
    listenerToPut.registrationName,
    listenerToPut.listener,
  );
}
````

`EventPluginHub` does do something hub will do.

```` Javascript
// EventPluginHub
putListener: function(inst, registrationName, listener) {
    invariant(
      typeof listener === 'function',
      'Expected %s listener to be a function, instead got type %s',
      registrationName,
      typeof listener,
    );

    var key = getDictionaryKey(inst);
    var bankForRegistrationName =
      listenerBank[registrationName] || (listenerBank[registrationName] = {});
    bankForRegistrationName[key] = listener;

    var PluginModule =
      EventPluginRegistry.registrationNameModules[registrationName];
    if (PluginModule && PluginModule.didPutListener) {
      PluginModule.didPutListener(inst, registrationName, listener);
    }
}
````

See, EventPluginHub stores event listener in listenerBank.

> **How Is Event Dispatched**

You have already seen that `ReactEventListener.dispatchEvent.bind(null, topLevelType)` is added as event callback when it's in the phase of `trapBubbledEvent`. So in file `ReactEventListener`, `dispatchEvent` will be the entrance of event dispatch.

````Javascript
// In ReactEventListener.js
dispatchEvent: function(topLevelType, nativeEvent) {
    if (!ReactEventListener._enabled) {
      return;
    }

    var bookKeeping = TopLevelCallbackBookKeeping.getPooled(
      topLevelType,
      nativeEvent,
    );
    try {
      // Event queue being processed in the same cycle allows
      // `preventDefault`.
      ReactUpdates.batchedUpdates(handleTopLevelImpl, bookKeeping);
    } finally {
      TopLevelCallbackBookKeeping.release(bookKeeping);
    }
}
````

We still batch process event as you can see `ReactUpdates.batchedUpdates(handleTopLevelImpl, bookKeeping);` is defined within dispatchEvent.
Therefore, `handleTopLevelImpl`  is the core.

````Javascript
function handleTopLevelImpl(bookKeeping) {
  var nativeEventTarget = getEventTarget(bookKeeping.nativeEvent);
  var targetInst = ReactDOMComponentTree.getClosestInstanceFromNode(
    nativeEventTarget,
  );

  // Loop through the hierarchy, in case there's any nested components.
  // It's important that we build the array of ancestors before calling any
  // event handlers, because event handlers can modify the DOM, leading to
  // inconsistencies with ReactMount's node cache. See #1105.
  var ancestor = targetInst;
  do {
    bookKeeping.ancestors.push(ancestor);
    ancestor = ancestor && findParent(ancestor);
  } while (ancestor);

  for (var i = 0; i < bookKeeping.ancestors.length; i++) {
    targetInst = bookKeeping.ancestors[i];
    ReactEventListener._handleTopLevel(
      bookKeeping.topLevelType,
      targetInst,
      bookKeeping.nativeEvent,
      getEventTarget(bookKeeping.nativeEvent),
    );
  }
}
````

The important thing in `handleTopLevelImpl` is collecting event target's ancestors and saving them locally. That's why React Synthetic Event has its own bubbling system.

#### **CONTINUE**

`ReactEventListener._handleTopLevel` actually is using `ReactBrowserEventEmitter`'s handleTopLevel, let check that out:

````Javascript
handleTopLevel: function(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
  ) {
    var events = EventPluginHub.extractEvents(
      topLevelType,
      targetInst,
      nativeEvent,
      nativeEventTarget,
    );
    runEventQueueInBatch(events);
}
````

It's `crystal` clear that handleTopLevel does two things:
1. Construct React Synthetic Event.
2. Batch run events.

> **How Synthetic Event Is Constructed?**

````Javascript
extractEvents: function(
    topLevelType,
    targetInst,
    nativeEvent,
    nativeEventTarget,
  ) {
    var events;
    var plugins = EventPluginRegistry.plugins;
    for (var i = 0; i < plugins.length; i++) {
      // Not every plugin in the ordering may be loaded at runtime.
      var possiblePlugin = plugins[i];
      if (possiblePlugin) {
        var extractedEvents = possiblePlugin.extractEvents(
          topLevelType,
          targetInst,
          nativeEvent,
          nativeEventTarget,
        );
        if (extractedEvents) {
          events = accumulateInto(events, extractedEvents);
        }
      }
    }
    return events;
}
````

Due to limited time, I will write a new post to talk detailed process while constructing react synthetic events. But here we need to be clear about some points:
1. When constructing its Synthetic Event, React use a Object pool mechanism which will dramatically reduce the time spent of object creation and destroy, so as to largely improve its performance.
2. Event is based on templates. Here, `EventPluginRegistry` contains five plugins: SimpleEventPlugin, EnterLeaveEventPlugin, ChangeEventPlugin, SelectEventPlugin, BeforeInputEventPlugin. They have their own processing respectively.

> **After Synthetic Event Is Constructed**

Then we `runEventQueueInBatch(events);`. Again, I will upload a new post to elaborate on this topic.