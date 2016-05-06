# Using Channels

Until now We've used the `take` and `put` effects to communicate with the Redux Store. Channels generalize those Effects to communicate with external event sources or between Sagas themselves. They can also be used to queue specific actions from the Store.

In this section, we'll see:

- How to use the `yield actionChannel` Effect to buffer specific actions from the Store.

- How to use the `eventChannel` factory function to connect `take` Effects to external event sources.

- How to create a channel using the generic `channel` factory function and use it in `take`/`put` Effects to
communicate between two Sagas.

## Using the `actionChannel` Effect

Let's review the canonical example:

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const {payload} = yield take('REQUEST')
    yield fork(handleRequest)
  }
}

function* handleRequest(payload) { ... }
```

The above example illustrates the typical *watch-and-fork* pattern. The `watchRequests` saga is using `fork` to avoid non-blocking and thus not missing any action from the store. A `handleRequest` task is created on each `REQUEST` action. So if there are many actions fired at a rapid race there can be many `handleRequest` tasks executing on parallel.

Imagine now that our requirement is as follow: we want to process `REQUEST` only one by one. Meaning if we have at a moment say four actions, we want to handle the 1st `REQUEST` action, then only after finishing processing the action we process the 2nd action on so on...

So what we want is to *queue* all non processed actions, and once we're done with processing the current request, we get the next message from the queue.

The library provides a little helper Effect `actionChannel` which can handle this stuff for us. Let's see how we can rewrite the previous example with it:

```javascript
import { take, actionChannel, call, ... } from 'redux-saga/effects'

function* watchRequests() {
  // 1- Create a channel for request actions
  const requestChan = yield actionChannel('REQUEST')
  while (true) {
    // 2- take from the channel
    const {payload} = yield take(requestChan)
    // 3- Note that we're using a blocking call
    yield call(handleRequest)
  }
}

function* handleRequest(payload) { ... }
```

The first thing is to create the action channel. We use `yield actionChannel(pattern)` where pattern is interpreted using the same rules we mentioned previously with `take(pattern)`. The difference between the 2 forms is that `actionChannel` **can buffer incoming messages** if the Saga is not yet ready to take them (e.g. blocked on an API call).

Next thing is the `yield take(requestChan)`. Besides usage with a `pattern` to take specific actions from the Redux Store, `take` can also be used with channels (above we created a channel object from specific Redux actions). The `take` will block the Saga until a message is available on the channel. The take may also resume immediately if there is a message stored in the underlying buffer.

The important thing to note is how we're using a blocking `call`. The Saga will remain blocked until `call(handleRequest)` returns. But meanwhile, if other `REQUEST` actions are dispatched while the Saga is still blocked, they will queued internally by `requestChan`. When the Saga resumes from `call(handleRequest)` and executes the next `yield take(requestChan)`, the take will resolve with the queued message.

By default, `actionChannel` buffers all incoming messages without limit. If you want a more control over the buffering, you can supply a Buffer argument to the effect creator. The library provides some common buffers (none, dropping, sliding) but you can also supply your own buffer implementation. See API docs for more details.

For example if you want to handle only the most recent five items you can use:

```javascript
import { buffers } from 'redux-saga'
import { actionChannel } from 'redux-saga/effects'

function* watchRequests() {
  const requestChan = yield actionChannel('REQUEST', buffers.sliding(5))
  ...
}
```

## Using the `eventChannel` factory to connect to external events

Like `actionChannel` (Effect), `eventChannel` (a factory function, not an Effect) creates a Channel for events but from event sources other than the Redux Store.

This simple example creates a Channel from an interval:

```javascript
import { eventChannel, END } from 'redux-saga'

function countdown(seconds) {
  return eventChannel(listener => {
      const iv = setInterval(() => {
        secs -= 1
        if (secs > 0) {
          listener(secs)
        } else {
          // this causes the channel to close
          listener(END)
          clearInterval(iv)
        }
      }, 1000);
      // The subscriber must return an unsubscribe function
      return () => {
        clearInterval(iv)
      }
    }
  )
}
```

The first argument `eventChannel` is a *subscriber* function. The rule of the subscriber is to initialize the external event source (above using `setInterval`), then routes all incoming events from the source to the channel by invoking the supplied `listener`. In the above example we're invoking `listener` on each second.

Note also the invocation `listener(END)`. We use this to notify any channel consumer that the channel has been closed, meaning no other message will come through this channel.

Let's see how we can use this channel from our Saga. This example is taken from the cancellable-counter example in the repo.

```javascript
import { take, put, call } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// creates an event Channel from an interval of seconds
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {    
    while (true) {
      // take(END) will cause the saga to terminate by jumping to the finally block
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    console.log('countdown terminated')
  }
}
```

So the Saga is yielding a `take(chan)`. This cause the Saga to block until a message is putted on the channel. In our example above, it corresponds to when we invoke `listener(secs)`. Note also we're executing the whole `while (true) {...}` loop inside a `try/finally` block. When the interval terminates, the countdown function closes the event channel by invoking `listener(END)`. Closing a channel has the effect of terminating all Sagas blocked on a `take` from that channel, in our example, terminating the Saga will cause it to jump to its `finally` block (if provided, otherwise the Saga simply terminate).

The subscriber returns an `unsubscribe` function. This is used by the channel to unsubscribe before the event source complete. Inside a Saga consuming messages from an event channel, if we want to *exit early* before the event source complete (e.g. Saga ha been cancelled) you can call `chan.close()` to close the channel and unsubscribe from the source.

For example, we can make our Saga support cancellation:

```javascript
import { take, put, call, cancelled } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// creates an event Channel from an interval of seconds
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {    
    while (true) {
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    if (yield cancelled()) {
      chan.close()
      console.log('countdown cancelled')
    }    
  }
}
```

> Note: messages on an eventChannel are not buffered by default. You have to provide a buffer to the eventChannel factory in order to specify buffering strategy for the channel (e.g. `eventChannel(subscriber, buffer)`). See the API docs for more info.

### Using channels to communicate between Sagas

Besides actions channels and event channels. You can also directly create channels which are not connected to any source by default. You can then manually `put` on the channel. This is handy when you want to use a channel to communicate between sagas.

To illustrate, let's review the former example of request handling.

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const {payload} = yield take('REQUEST')
    yield fork(handleRequest)
  }
}

function* handleRequest(payload) { ... }
```

We saw that the watch-and-fork pattern allows to handle multiple requests simultaneously, without limit on the number of worker tasks executing in parallel. Then we used the `actionChannel` effect to limit the concurrency to one task at a time.

So let's say that our requirement is to have a maximum of three tasks executing in the same time. When we get a request and there are less than three tasks executing, we process the request immediately, but if we have already three tasks executing, we queue the task and wait for one of the three *slots* to be free.

Below an example of a solution using channels

```javascript
import { channel } from 'redux-saga'
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  // create a channel to queue incoming requests
  const chan = yield call(channel)

  // create 3 worker 'threads'
  for (var i = 0; i < 3; i++) {
    yield fork(handleRequest, chan)
  }

  while (true) {
    const {payload} = yield take('REQUEST')
    yield put(chan, payload)
  }
}

function* handleRequest(chan) {
  while (true) {
    const payload = yield take(chan)
    // process the request
  }
}
```

In the above example, we create a channel using the `channel` factory. We get back a channel which by default buffers all message we put on it (unless there is a pending taker, in which the taker is resumed immediately with the message).

The `watchRequests` saga then forks three worker sagas. Note the created channel is supplied to all forked sagas. `watchRequests` will use this channel to *dispatch* work to the three worker sagas. On each `REQUEST` action the Saga will simply put the payload on the channel. The payload will then be taken by any *free* worker. Otherwise it will be queued by the channel until a worker Saga is ready to take it.

All the three workers run a typical while loop. On each iteration, a worker will take the next request, or will block until a message is available. Note that this mechanism provides an automatic load-balancing between the 3 workers. Rapid workers are not slowed down by slow workers.