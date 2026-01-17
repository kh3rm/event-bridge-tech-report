# An Event Bridge - A simple, neutral, stateless relayer of authoritative event streams


## Introduction

The inspiration for this technical report grew out of an unassuming supporting structure that we made use of in our project, which proved very effective, reliable and solid, despite being made up of only a few rows of code.

### Purpose of This Report

This report aims to highlight this small, under-appreciated architectural pattern that proved very effective, precisely because it wisely chooses to do very little.

The construct, that I have chosen to refer to as an **_Event Bridge_**, helped us with a very practical need: to deliver clean, simple, event-driven [1] updates from an internal system container to a frontend client without extending ownership, responsibility, or coupling. 

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

        # Real-time push for the live map updates
        self.r.publish("scooter:state:tick", encoded)
```


Each tick therefore emits a complete, self-contained snapshot of the scooter at that moment. These updates are already resolved and self-contained by the time they are published.

A corresponding Redis subscriber was established in the main Node/Express-based backend container, where these updates become available as a real-time event stream.

What remained: delivering those updates to the admin frontend container and its map view, which needs to render and continuously update scooter markers reflecting position, speed, status, and battery level.

What was needed at this point was not additional logic or state handling, but a simple forwarding service to the frontend - a bridge. For this purpose, the choice fell on classic, fast, reliable, minimalistic WebSockets (WSS).



### Bridge-side forwarding

On the backend, the bridge subscribes to the scooter:state:tick Redis Pub/Sub channel. For every message received on that channel, the bridge forwards the message verbatim to all currently connected frontend WebSocket clients.

server.js (backend)

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

The initial plan was to write this report under the heading **_Socket Bridge_**. While accurate enough as far as describing the implementation, that framing places slightly misplaced emphasis on the transport mechanism rather than on the architectural role.

Seen through this lens, the structure is thus better and more generally and accurately conceptualized as an **_Event Bridge_**.

The WebSockets specifically, although very useful in this case, and a pleasure to work with, are really incidental. The real value lies in the separations and boundaries that this component and setup enforce.

What makes this structure so effective is, again, how clearly it separates responsibilities. The simulator reconciles state and emits events. The bridge relays those events without inspecting or reshaping them. The frontend consumes the events to populate and continuously update its scooter markers, reflecting the current state of each scooter.

By sitting cleanly at the handoff point between the authority and the consumer, the **_Event Bridge_** keeps the flow of data extremely simple, easy to understand, and hard to misconfigure.

## Definition
An **_Event Bridge_** is a stateless component that forwards an authoritative stream of events from an internal source of truth to downstream consumers without interpretation, manipulation, validation, persistence, or ownership.


### Why _Event_ Bridge?
An **_Event Bridge_** works because events are fully resolved at the moment they are produced. They represent completed decisions - immutable facts - not negotiable state.
This removes the need for any reconciliation, version tracking, or downstream validation.

Because of this, the bridge can remain pristine as a simple conduit. It simply forwards events agnostically. If a client consumer misses an event, its view and its update may be incomplete, but system correctness is unaffected because authority remains upstream.

This approach however breaks down for mutable data. Bridging state would require the intermediary to track updates, resolve conflicts, and decide which version is correct - effectively assuming partial ownership. The **_Event Bridge_** avoids all of these pesky problems by operating strictly on events, not state.

## Transport Agnosticism

An **_Event Bridge_** is characterized by its purpose, not its transport layer. WebSockets are a great option for real-time bidirectional links, but Server-Sent Events, HTTP streams, or durable broker relays (e.g., Kafka, RabbitMQ, etc.) can serve similar roles.

The key is maintaining the clear boundary between internal event flow and client delivery. The transport type may change, but responsibility does not.

## Question
### _Neat. But why not just skip this extra step altogether and connect the frontend(s) directly to the Redis Pub/Sub-pipeline?_

Redis Pub/Sub is designed for internal system communications, not for frontend-facing interfaces.

Connecting browser-based frontends directly to Redis introduces several concrete problems:


* **Security risk** - Redis credentials would need to leave the server, making them inspectable and thus exposing internal infrastructure. [2]


* **Protocol mismatch**: Redis speaks via its own TCP-based protocol, which browsers cannot and should not connect to directly. [3]


* **Tight coupling**: Frontends would become dependent on Redis-specific details such as channel names and retry behavior, making backend changes immediately disruptive.


* **Operational fragility**: Clients would need Redis-specific reconnect and error-handling logic.


* **Scaling risk**: Redis is fundamentally designed for trusted server-to-server traffic, not large numbers of long-lived browser connections. [4]


The Event Bridge avoids all of this. It terminates the internal Redis connection and exposes the same event stream through a single, client-friendly endpoint. Internal details stay hidden, the bridge remains minimalistic and stateless.

### Key Point to drive home

Internal buses are not interfaces.
An **_Event Bridge_** ends internal event transport and deliberately exposes a clean, safe client-facing stream.


### Failure and Error Handling
Because it holds no state and assumes no authority, the **_Event Bridge_** fails narrowly and predictably. It may stop the delivery of events, but it does not corrupt state or introduce inconsistencies.
Availability may degrade, updates may suffer - correctness does not.


## Conclusion
The **_Event Bridge_** is a deliberately modest and minimalistic structure for safely exposing authoritative event streams beyond internal boundaries. It applies naturally to event-driven structures or sub-structures where frontend clients need to be fed in a safe, decoupled and timely matter.

Its strength comes from its restraint. By staying true to its role as a neutral conduit and relayer, it removes entire classes of complexity, coupling, and failure.

The result is an exceedingly useful, clear, succinct, understandable, clean structure that is easy to reason about, operate and adjust.


## Addendum: Terminology

Structures resembling the **_Event Bridge_** commonly appear as adapters, gateways, or edge relays.
The term is introduced here not for novelty, but solely to better highlight this specific role: a simple handoff point between an internal event stream and external consumers.
Having a distinctive term helps one reason about this boundary independent of specific transport or implementation details.


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

