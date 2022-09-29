# Objectives

Specify a minimal NAT traversal library for UDP.

# Specification

## Constants

### `LOCAL_PORT`
The UDP port to bind to.

```c
const uint LOCAL_PORT = 3456;
```

### `TEST_PORT`

The read only port to bind to that will accept inbound data and help determine if a NAT is static.

```c
const uint TEST_PORT = 3457;
```

### `BDP`
The delay between packets sent for birthday paradox connection 10ms means 100 packets per second.

```c
const uint BDP = 10;
```

### `BDP_MAX_PACKETS`
The maximum number of packets to use when employing the birthday paradox strategy. On average, about ~255 packets are sent per successful connection (giving up after 1000 packets
means 97% of attempts are successful. It is necessary to give up at some point because the other side might not have done anything, or might have crashed, etc).

```c
const uint BDP_MAX_PACKETS = 1000;
```

### `CONNECTING_MAX_TIME`
The time that we expect a new connection to take. Do not start another new connection attempt within this time, even if we havn't received a packet yet.

```c
const uint CONNECTING_MAX_TIME = BDP * BDP_MAX_PACKETS;
```

### `KEEP_ALIVE_TIMEOUT`
100 byte keepalive packet 120 times an hour 24 hours is 0.288 mb a day per peer.

```c
const uint KEEP_ALIVE_TIMEOUT = 29_000;
```

## Data Structures

### Nat

```c
enum Nat {
  Easy,
  Hard,
  Static
}
```

### PeerIdentity

```c
struct PeerIdentity {
  string id, // a unique identifier for this peer
  string address, // a valid IP address
  uint port // a valid port number
}
```

### PeerStates

```c
struct PeerStates {
  float Forgotten = 5,
  float Missing = 3.0,
  float Inactive = 1.5,
  float Active = 0.0
}
```

### Config

```c
struct Config {
  localPort: LOCAL_PORT,
  testPort: TEST_PORT,
  bdp: BDP,
  nat: null,
  bdpMaxPackets: BDP_MAX_PACKETS,
  connecting: CONNECTING_MAX_TIME,
  keepAlive: KEEP_ALIVE_TIMEOUT,
  introducerA: PeerIdentity iA,
  introducerB: PeerIdentity iB
}
```

### ArgsAddPeer

```c
struct ArgsAddPeer {
  string id, // the unique identity of the peer
  string address, // the ip address of the peer
  uint port, // the numeric port of the peer
  Nat nat, // the nat type of the peer
  uint outport, // the outgoing ephemeral port of the peer
  uint restart, // timestamp of the last restart
  uint ts, // timestamp
  bool isIntroducer // if this peer static
}
```

### ArgsBind

```c
struct ArgsBind {

}
```

### ArgsConnect

```c
struct ArgsConnect {

}
```

## Classes

A peer MUST, in some way, implement at least these methods and properties. Regardless of how you implement a peer, it is impossible to demonstrate the reliability of this solution when deployed. So it is necessary to specify a [TLA+][3] spec as well as run the code on top of a network simulation. Tests that demonstrate [`Safety`][0] and [`Liveness`][1] properties will need to override the `[init, onMessage, localAddress, timer, send]` methods and properties so that they can be run synchronously in a simulation.

```c
class Peer {
  bool isIntroducer = false; // set to true if peer has a static IP address
  string localAddress; // deteremind by checking the network interfaces
  uint localPort; // set in the configuration
  string publicAddress; // set when a pong is received
  uint publicPort; // set when a pong is received

  void addPeer (ArgsAddPeer args);
  void connect (ArgsConnect args);
  void constructor (Config config, uint timestamp);
  void bind (ArgsBind args);
  void calculateNat (ArgsCalculateNat args);
  void requestNat (ArgsRequestNat args);
  void init (uint timestamp);
  void intro ();
  void onIntro (ArgsMessage args);
  void onMessage (Any data, string address, uint port, uint timestamp);
  void onPing (ArgsMessage args);
  void onPong (ArgsMessage args);
  void onSpin (ArgsMessage args);
  void ping (ArgsPing args);
  void retryPing ();
  void timer (uint delay, uint repeat, function<void(uint timestamp)> cb);
}
```

## Messages

### Ping

```c
struct Ping {
  type: string, // the type of the message
  id: string, // the unique id of the sending-peer
  nat: Nat, // the typeof the
  restart: uint // a unix timestamp specifying uptime of the sending-peer
}
```

### Pong

```c
struct Pong {
  type: string, // the type of the message
  id: string, // the unique id of the sending-peer
  address: string, // a string representation of the ip address of the sending-peer
  port: uint, // a numeric representation of the port of the sending-peer
  nat: Nat, // the typeof the
  restart: uint, // a unix timestamp specifying uptime of the sending-peer
  ts: uint // a unix timestamp specifying the time the ping message was received
}
```

## States

This section outlines the states of the program.

### Initial

#### Execution
- An instance of the Peer class is constructed
  - At least two UDP ports are bound, namely `.localPort` and `.staticDetectPort`
    - Messages are parsed with JSON as the default encoding
  - An interval will poll for network interface changes
    - if there is no `keepAlive` specified in the config the function returns
    - An additional interval is started after `keepAlive` and repeats every `keepAlive` seconds
      - If `currentTime` - `lastPongReceived` > `keepalive` * `5`, the peer is Forgotten
      - If `currentTime` - `lastPongReceived` > `keepalive` * `3`, the peer is Missing
      - If `currentTime` - `lastPongReceived` > `keepalive` * `1.5`, the peer is Inactive
      - Otherwise the peer is considered Active
    - if there is an interface change, the NAT type is re-evaluated and the function returns
    - if the time ellapsed is greater than a single cycle of the interval, a wakeup event is emitted
    - if the `Nat` type has become unknown the NAT type is re-evaluated

### NAT Evaluation

#### Static Nat

#### Easy Nat

The nat allows a device to use the same port to communicate with other hosts. If you are on an easy NAT, you just have to find out what port you have been given and then other peers will be able to message you on that port.

#### Hard Nat

The nat assigns different (probably random) ports for every other host you communicate with. Since a port cannot be reused, connecting as a hard nat is more complicated. Multiple approaches need to be used, first port mapping protocols need to be tried, failing that, the birthday paradox must be used. |

#### Execution

A router's routing table or firewall may drop "unsolicited" packets. So simply binding a port and waiting for connections won't work.

However a router (even one with a firewall) can be cooerced into accepting packets in a perfectly safe way.

A NAT check requires a peer (`P0`) to initially bind two ports, `defaultPort` and `testPort`. In addition, two introducers (`I0`, `I1`) are required, they should reside on separate static peers outside the NAT being tested.

- The `.publicAddress` and `.nat` properties of the `Peer` are set to `null`
- `P0` sends `Ping` to `I0` and `I1`.
- `I0` and `I1` should respond by sending `Pong` to `P0` and the message includes the NAT type and public IP and ephemeral port.
- `I0` and `I1` also respond by sending a message to `P0` on the `testPort`.
  - If `P0` receives a message on `testPort` we know that our NAT type is Static
- Finally, `P0` must calculate the nat type based on the data collected so far

### Receive Pong

#### Execution

- A message of type `Pong` is received
  - The peer is added to a local list of known peers
  - The peer updates a property on itself called `pong` which contains the `timestamp` of the event, as well as the `address` and `port` from the data received.
  - The peer updates a property on itself to track the time t

### Receive Ping

#### Execution

- A message of type `Ping` is sent to a `Peer`
  - The `Peer` will try to respond with a message of type `Pong`

### Receive Intro

### Receive Spin

# Credit

This work is derived from the work of Bryan Ford (<baford@mit.edu>), Pyda Srisuresh (<srisuresh@yahoo.com>) and Dan Kegel (<dank@kegel.com>) who published ["Peer-to-Peer Communication Across Network Address Translators"][2].

[0]:https://lamport.azurewebsites.net/tla/proving-safety.pdf
[1]:https://lamport.azurewebsites.net/pubs/liveness.pdf
[2]:https://pdos.csail.mit.edu/papers/p2pnat.pdf
[3]:https://www.microsoft.com/en-us/research/uploads/prod/2018/05/book-02-08-08.pdf
