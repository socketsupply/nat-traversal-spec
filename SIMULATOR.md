# Network Simulator

## Motivation

Developing distributed systems is challenging, because distributed programs are not sequential. A network of nodes operates by sending asyncronous messages to each other and those messages may be dropped or arrive out of order. This means that the order of events in a system is at best partially ordered, and correctness must be defined over a set of possible partial orders. A correct distributed system must still achive it's goals given any possible message ordering.

Testing such a system simply by running it is extremely complicated because it's the diversity of real world networks and devices that produces the unpredictable behavior, and an easy to set up test rig would just be a collection of cloud servers - which would likely behave much more reliably, so wouldn't be very representative!

More, there can be subtle bugs that happen infrequently on any given node, but in a network with many nodes, happen to some node frequently - reproducing such a bug on a developer machine might take months, so it would be extremely difficult to manually debug.

Instead, we use perform testing in a _network simulation_. This gives the possibility to model network behavior in a deterministic way. A simplified abstract model of the network is used. It would be very difficult to make a very realistic network, both difficult to define, and difficult to argue that it is realistic. Instead we make a network model that is auguably _worse_ than realistic. For example, instead of predictable latency, the next message to be delivered is just random, and instead of dropping messages based on router buffer over-capacity, simply drop packets with a configurable uniform random probability.

By using deterministic random message delivery, we sample the space of possible orderings. It would might be better to exaustively explore the ordering space, but much simpler to sample it. By running a test many hundreds or thousands of times we can detect bugs that happen infrequently. Then, by replaying those test runs with a deterministic seed, test failures can be reproduced instantly, and debugged.

## Event

An event is a tuple of a `(ts, fn)` some code to be executed at a specific time in the simulation's execution.
This is used to model timers scheduled to run at specific times, and also used to model message delivery my using random times in the near future.
 
## Queue


The Events go into a single Queue that the current state of the simulation. Events are always removed smallest `.ts` value first, but tuples may be added in random order as long as `ts` is greater than the last event that has been processed. (see `Queue.ts` property. Processing an event may result in additional events being inserted into the Queue.

`add(ts, fn)` adds an event tuple, with `fn` to be processed at specific time `ts`.

`ts` a property of the Queue which is set to the `ts` of the event to be processed. It is set immediately before the event is processed.

`drain(ts)` processes events in the queue, until there are either no more events, or the remaining events have a scheduled time greater than `ts`. All events are processed in order of their attached `ts`.

## Nodes

The test node represents a device on the network.
Once a node has been added to a network, it may send and receive messages.
Nodes also have a "sleeping" state.
Node sleep is used to model laptops suspending and mobile apps being backgrounded,
where the program state of the node is maintained but they do not respond to events.

`onMessage(msg, addr, port)` function to represent receiving a message. This method is overridden and used to model the specific behaviour of this particular node.
It is called with the message object, the addr of the sender, and the port sent to. (note, because p2p holepunching sometimes requires binding many ports we don't bother to represent binding each socket as a separate object, but just sending to and from ports)

`send(msg, to_addr, from_port)` send a message to `to_addr` from `from_port`.

`init(ts)` a method called by the simulation when the simulation starts. Clients and p2p protocols do not passively wait for connections, but actively seek out peers in the network, so must be given an opportunity to start doing things.

the `timer(delay, repeat, fn)` method schedules events. if `repeat` is zero, the event runs a single time. If `delay` is zero, the event occurs immediately (before timer() returns) then at the `repeat` interval.

If the node is asleep, the event not processed, and is added to the `awaken` list.
The awaken list represents events that were not processed because the node was asleep.
When the node comes out of sleep, the events are processed until the list is empty or one of the events causes the node to sleep again. If a repeating interval is scheduled and a node sleeps through several iterations of the interval, just a single event is processed at the time the sleep ends. This reflects the behavior of the javascript `setInterval` method. It is the users responsibility to detect if it has actually been several cycles, if necessary.

## Address

ipv4 address, 4 byte integer in dotted decimal `a.b.c.d` "ip address" notation.

## Network

A network is a derivative of Node, and has a map of `subnet` of type Address->Node. (because an Network is a node, a network can map to subnetworks. This is used to model Nat behaviour and private networks, see: Nat)

`add(address, node)` add an address to the network. If the network has been initialized then call `init(ts)` on the node.

`remove(node)` remove node from the network's address map.

## Nat

derivative of Network used as a base to model Network Address Translation

adds {TTL, map, unmap, hairpinning} properties to Network.
`TTL` is a the number of miliseconds before an entry in the firewall should be expired.
`map` is a mapping of internal to external ports.
`unmap` is the reverse of map.
`hairpininng` is a boolean, if true, messages from internal nodes may address the public address of the network and they will be delivered to another internal node based on the port. (this behaviour is not usually supported most real world nats)

