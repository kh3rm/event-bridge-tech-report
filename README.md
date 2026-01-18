# An Event Bridge - A neutral relayer of events across the internal–client boundary

## Introduction

The inspiration for this technical report grew out of an unassuming supporting structure used in our project, which proved highly effective, reliable, and solid, using a remarkably small amount of code.

### Purpose of This Report

This report aims to highlight this small, under-appreciated architectural pattern, which derives its strength from intentionally doing very little.

The construct, that I have chosen to refer to as an **_Event Bridge_**, helped us with a very practical need: to deliver clean, simple, event-driven [1] updates from an internal system container to a frontend client, acting solely as a neutral forwarding layer for events.

> **Note**  
> The following sections describe a concrete implementation based on Redis Pub/Sub and WebSockets. These technologies are used to ground the discussion in a real example, but they are not defining characteristics of the Event Bridge pattern itself. The architectural properties described do not depend on Redis or WebSockets specifically, but on the presence of a suitable, functionally equivalent event delivery mechanism.

## Example implementation

In our project’s multi-container setup, the simulation container is where scooter instances live and from which all the scooter-related activity eminates. On every simulation tick - update interval - each scooter emits its current state.

### Simulator-side emission

On each simulation tick, the simulator constructs a payload representing the full current state of a scooter and publishes it. The logic looks as follows:

The simulator builds a payload object containing the scooter identifier, position, battery level, status, and speed. This payload is encoded as JSON, and published on the Redis Pub/Sub channel scooter:state:tick.

scooter.py:
```python
    def publish(self):
        if not self.rbroadcast:
            return
        payload = {
            "id": self.id,
            "lat": round(self.lat, 7),
            "lng": round(self.lng, 7),
            "bat": round(self.battery, 1),
            "st": self.status,
            "spd": self.speed_kmh,
        }
        self.rbroadcast.broadcast_state(payload)
```
rbroadcast.py:
```
...

        self.r.publish("scooter:state:tick", encoded)
```


Each tick therefore emits a complete, self-contained snapshot of the scooter at that moment. These updates are already resolved and self-contained by the time they are published.

A corresponding Redis subscriber was established in the main Node/Express-based backend container, where these updates become available as a real-time event stream.

What remained: delivering those updates to the admin frontend container and its map view, that should render and continuously update scooter markers reflecting position, speed, status, and battery level.

Nothing more needed to be decided or computed. All that was required was a component capable of forwarding the Redis event stream to the frontend - a bridge. For this purpose, the choice fell on classic, fast, reliable, minimalistic WebSockets (WSS).

### Bridge-side forwarding

On the backend, the bridge subscribes to the scooter:state:tick Redis Pub/Sub channel. For every message received on that channel, the bridge forwards the message verbatim to all currently connected frontend WebSocket clients.

socket/socketBridge.js (backend)

```
    const clients = new Set();

    wss.on('connection', (ws) => {
      clients.add(ws);


    redisSubscriber.subscribe('scooter:state:tick', 'rental:completed');

    redisSubscriber.on('message', (channel, message) => {
    for (const client of clients) {
        if (client.readyState === client.OPEN) {
        client.send(message);
        }
    }
    });
```

*This alone handles it all.*

The bridge does not inspect the payload, validate it, transform it, or store it. It simply terminates the internal Redis connection and re-exposes the same event stream to frontend consumers over a client-friendly transport.

There is no session handling, no validation, no transformation, and no persistence. The component listens and forwards, without retaining or acting on the data in any way.

## Strenghts

What stood out most was not the mechanics of this setup, but how clearly the responsibilities fell into place.

Canonical scooter status is persisted and updated in the database, while the simulator reconciles that state and emits the authoritative event stream consumed downstream. Because updates are emitted only after these changes have been decided and reconciled, they arrive as ordered, self-contained events.

The forwarding layer never needs to reconcile, compute, or interpret anything. It remains neutral, serving only as a clean handoff point between backend and frontend.

The **Event Bridge** narrows the responsibility surface for each team. Backend ownership ends at event emission, frontend ownership begins at consumption, and the boundary between them remains small and stable.

The initial plan was to write this report under the title **_Socket Bridge_**. While accurate enough as far as describing the given implementation, that framing places slightly misplaced emphasis on the transport mechanism rather than on the architectural role.

Seen through this lens, the structure is better and more generally and accurately conceptualized as an **_Event Bridge_**.

The WebSockets in this example, although very useful, and a pleasure to work with, are really incidental. The real value lies in the separations and boundaries that this component and setup enforce.

What makes this structure so effective is, again, how clearly it separates the responsibilities. The simulator reconciles state and emits events. The bridge relays those events without inspecting or reshaping them. The frontend consumes the events to populate and continuously update its scooter markers, reflecting the latest snapshot received of each scooter.

By sitting cleanly at the handoff point between the authority and the consumer, the **_Event Bridge_** keeps the flow of data extremely simple, easy to understand, and hard to misconfigure.

## Definition
An **_Event Bridge_** forwards events across the internal–client boundary without interpreting, transforming, or acting on them.


### Why _Event_ Bridge?
An **_Event Bridge_** works because events are fully resolved at the moment they are produced. They represent completed decisions - immutable facts - not negotiable state.
This removes the need for any reconciliation, version tracking, or downstream validation.

Because of this, the bridge can remain pristine as a simple conduit. It simply forwards events agnostically. Missed events may result in incomplete client views, but authoritative state and decisions remain upstream.

This approach however breaks down for mutable data. Bridging state would require the intermediary to track updates, resolve conflicts, and decide which version is correct - effectively assuming partial ownership. The **_Event Bridge_** avoids all of these pesky problems by operating strictly on events, not state.

## Transport Agnosticism

An **_Event Bridge_** is defined by its purpose, not its transport layer. WebSockets are a great option for real-time, low-latency event delivery to clients, but Server-Sent Events, HTTP streams, or durable broker relays (e.g., Kafka, RabbitMQ, etc.) can serve similar roles.

The key is maintaining the clear boundary between internal event flow and client delivery. The transport type may change, but responsibilities do not.

## Question
### _Neat. But why not just skip this extra step altogether and connect the frontend(s) directly to the Redis Pub/Sub-pipeline?_

Redis Pub/Sub assumes trusted, server-side peers and uses a native TCP protocol that browsers cannot directly handle. [2]

As a result, it simply can not and should not be consumed directly by frontend clients.

### Failure and Error Handling

If the **_Event Bridge_** fails, the event delivery to clients stops, but no state is lost, no decisions are reversed, and no reconciliation is required when it resumes. Clients may observe stale or incomplete views during the outage, which are then corrected automatically as new events arrive.

## Conclusion

Emitting events, interpreting them, relaying them, and consuming them are distinct and separate roles, and the Event Bridge deliberately confines itself only to the relay.

Its strength comes from its restraint. By staying true to its role as a neutral conduit and relayer, it avoids introducing additional complexity, coupling, or error-handling concerns of its own.

The result is an exceedingly useful, clear, intelligible, simple, and predictable structure that is easy to operate and adjust.

## Addendum: Terminology

Patterns resembling an Event Bridge frequently appear under names such as adapters, gateways, or broadcast relays.
In concrete terms, the example implementation aligns closely with a Redis broadcast relay, which forwards events emitted via Redis Pub/Sub to connected consumers.

The Event Bridge terminology is used not for novelty, but to support clear reasoning and discussion by establishing a consistent name for this architectural boundary throughout the text. It captures its role more broadly as a transport-agnostic interface between an internal event stream and external consumers, without coupling the design to Redis, WebSockets, or any other specific implementation.

## Author:
Herman Karlsson --- vteam 02 --- spark ⚡


## References

[1] Martin Fowler.  
*What Do You Mean by “Event-Driven”?*  
MartinFowler.com, 2017.  
https://martinfowler.com/articles/201701-event-driven.html

[2] Redis Documentation.  
*Redis Security.*  
Redis Ltd.  
https://redis.io/docs/latest/operate/oss_and_stack/management/security/

[3] MDN Web Docs.  
*WebSocket API.*  
Mozilla.  
https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API

[4] Redis Documentation.  
*Redis Client handling.*  
Redis Ltd.  
https://redis.io/docs/latest/develop/reference/clients/

