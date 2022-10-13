# Network Simulator

## Motivation

Developing distributed systems is challenging, because distributed programs are not sequential. A network of nodes operates by sending asyncronous messages to each other and those messages may be dropped or arrive out of order. This means that the order of events in a system is at best partially ordered, and correctness must be defined over a set of possible partial orders. A correct distributed system must still achive it's goals given any possible message ordering.

Testing such a system simply by running it is extremely complicated because it's the diversity of real world networks and devices that produces the unpredictable behavior, and an easy to set up test rig would just be a collection of cloud servers - which would likely behave much more reliably, so wouldn't be very representative!

More, there can be subtle bugs that happen infrequently on any given node, but in a network with many nodes, happen to some node frequently - reproducing such a bug on a developer machine might take months, so it would be extremely difficult to manually debug.

Instead, we use perform testing in a _network simulation_. This gives the possibility to model network behavior in a deterministic way. A simplified abstract model of the network is used. It would be very difficult to make a very realistic network, both difficult to define, and difficult to argue that it is realistic. Instead we make a network model that is auguably _worse_ than realistic. For example, instead of predictable latency, the next message to be delivered is just random, and instead of dropping messages based on router buffer over-capacity, simply drop packets with a configurable uniform random probability.

By using deterministic random message delivery, we sample the space of possible orderings. It would might be better to exaustively explore the ordering space, but much simpler to sample it. By running a test many hundreds or thousands of times we can detect bugs that happen infrequently. Then, by replaying those test runs with a deterministic seed, test failures can be reproduced instantly, and debugged.

## Queue

A simulation needs a single queue (heap data structure) to model events in the system. This is used to model message delivery and scheduled timers.

`add(ts, fn)` adds a function `fn` to be processed at specific time `ts`.

`ts` a property of the Queue which is set to the `ts` of the event to be processed. It is set immediately before the event is processed.

`drain(ts)` processes events in the queue, until there are either no more events, or the remaining events have a scheduled time greater than `ts`. All events are processed in order of their attached `ts`.







## Nodes

The test node represents a device on the network.
Once a node has been added to a network, it may send and receive messages.

## Network

A network is a derivative of Node, and has a map of address->node

