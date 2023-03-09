---
title: "The disgust of sequential processing of binary WebSocket messages in JavaScript"
date: 2023-03-09
---

Let's process binary WebSocket messages in the browser, in the order the server has sent them.
The [Writing WebSocket client applications][mdn-hello] MDN article is a reasonable place to start:

```javascript
const ws = new WebSocket('ws://foo.bar/endpoint')
ws.onmessage = (event) => {
  console.log(event.data)
}
```

If we were to receive [text messages][ws-dataframes], this'd be all we need.
However, with binary messages, we'll get:

```
Blob { size: 16, type: "" }
```

Ugh. So we need to extract the received bytes from this blob first.
Also note that, about `event.data` [MDN says][datatype]:

> The data sent by the message emitter; this can be any data type.

Therefore, a correct program should check if it really received a blob.
Unfortunately, [Blob][] does not provide a synchronous way to get to its data,
only through an asynchronous promise (and this will be a pain later):

```javascript
const ws = new WebSocket('ws://foo.bar/endpoint')
ws.onmessage = (event) => {
  event.data.arrayBuffer().then(function (ab) {
    console.log(ab)
  })
}
```

At this point, we have an [ArrayBuffer][]. It still does not allow reading its data,
we have to use another wrapper:

```javascript
const ws = new WebSocket('ws://foo.bar/endpoint')
ws.onmessage = (event) => {
  event.data.arrayBuffer().then(function (ab) {
    const dv = new DataView(ab)
    const i = dv.getUint32(0 /* offset */, true /* little endian */)
    console.log(i)
  })
}
```

Finally, we managed to read a 32 bit unsigned number from the binary payload.
The [DataView][] documentation says:

> The DataView view provides a low-level interface for reading and writing multiple number types in a binary ArrayBuffer, without having to care about the platform's endianness.

Obviously, we still need to care about the endinanness of the message!

## Ordering

So this was it? It wasn't that hard! Yes, and this solution is also wrong.
We wanted to process the messages in the order the server has sent them.
However, while we wait for the array buffer promise that fetches our data (see: `then`),
the onmessage handler yields to the runtime, that is free to schedule another task.
It is even free to schedule another onmessage invocation, and that promise might be ready earlier.
Therefore, it is possible, that we'll process messages out of order.
If the message rate is high enough, this possibility becomes a certainty.

One possible solution is to queue the promises, and process them in queue order, instead of resolution order:

```javascript
const Sequencer = () => {
  const xs = []
  let x0id = 0

  return {
    read: () => {
      if (xs[0] !== undefined) {
        x0id++
        return xs.shift()
      }
      return undefined
    },
    reserve: () => {
      xs.push(undefined)
      return x0id + xs.length - 1
    },
    write: (id, x) => {
      xs[id - x0id] = x
    }
  }
}

const processNextMessage = (q) => {
  for (let msg = q.read(); msg !== undefined; msg = q.read()) {
    const dv = new DataView(ab)
    const i = dv.getUint32(0 /* offset */, true /* little endian */)
    console.log(i)
  }
}

const q = Sequencer()
const ws = new WebSocket('ws://foo.bar/endpoint')
ws.onmessage = (event) => {
  const mi = messages.reserve()
  event.data.arrayBuffer().then(function (ab) {
    q.write(mi, ab)
    processNextMessage(q)
  })
}
```

## Conclusion

A tiny change in the requirements (binary messages instead of text messages)
inflicted a great deal of complexity on the correct implementation.
It is sad to see how seemingly simple tasks can turn out to be not so simple.


[mdn-hello]: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_client_applications
[ws-dataframes]: https://www.rfc-editor.org/rfc/rfc6455#section-5.6
[Blob]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[ArrayBuffer]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[DataView]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView
[datatype]: https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent/data
