# Objectives

Specify a minimal NAT traversal library for UDP.

# Specification

This implementation targets UDP. UDP is a [message-oriented][F0] [transport layer protocol][W1], ideal for talking to NATs because unlike TCP, it doesn't require a handshake to start communicating. It also delegates encryption and security responsibility to a higher level protocol or even the application layer, which in many cases is preferred.

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
enum NatType {
  Easy,
  Hard,
  Static
};
```

### PeerState

```c
struct PeerState {
  float Forgotten = 5;
  float Missing = 3.0;
  float Inactive = 1.5;
  float Active = 0.0;
};
```


### PeerIdentity

```c
struct PeerIdentity {
  string id; // a unique identifier for this peer
  string address; // a valid IP address
  uint port; // a valid port number
};
```

### Config

```c
struct Config {
  localPort: LOCAL_PORT;
  testPort: TEST_PORT;
  bdp: BDP;
  nat: null;
  bdpMaxPackets: BDP_MAX_PACKETS;
  connecting: CONNECTING_MAX_TIME;
  keepAlive: KEEP_ALIVE_TIMEOUT;
  introducerA: PeerIdentity iA;
  introducerB: PeerIdentity iB;
};
```

### ArgsAddPeer

```c
struct ArgsAddPeer {
  string id; // the unique identity of the peer
  string address; // the ip address of the peer
  uint port; // the numeric port of the peer
  NatType natType; // the nat type of the peer
  uint outport; // the outgoing ephemeral port of the peer
  uint restart; // timestamp of the last restart
  uint timestamp;
  bool isIntroducer; // if this peer static
};
```

### PongState

```c
struct PongState {
  uint timestamp;
  string address; // the ip address of the peer
  uint port; // the numeric port of the peer
};
```

### ArgsBind

```c
struct ArgsBind {

};
```

### ArgsConnect

```c
struct ArgsConnect {

};
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
  NatType natType; // this peer's NatType
  PongState pong; // the state of the last pong

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
  void onTest (ArgsMessage args);
  void ping (ArgsPing args);
  void retryPing ();
  void timer (uint delay, uint repeat, function<void(uint timestamp)> cb);
};
```

## Messages

### MsgPing

Generally sent as a "request" for a `MsgPong` message.

```c
struct MsgPing {
  string type = "ping"; // the type of the message
  string id; // the unique id of the sending-peer
  NatType natType;
  uint restart; // a unix timestamp specifying uptime of the sending-peer
};
```

### MsgPong

Generally sent as a "response" to a `MsgPing` message.

```c
struct MsgPong {
  string type = "pong"; // the type of the message
  string id; // the unique id of the sending-peer
  string address; // a string representation of the ip address of the sending-peer
  uint port; // a numeric representation of the port of the sending-peer
  NatType natType;
  uint restart; // a unix timestamp specifying uptime of the sending-peer
  uint timestamp; a unix timestamp specifying the time the ping message was received
};
```

### MsgTest

Sent to the `Config.testPort` of a peer as a response to a `MsgPing` message.

```c
struct MsgTest {
  string type = "test"; // the type of the message
  string id; // the unique id of the sending-peer
  string address; // a string representation of the ip address of the sending-peer
  uint port; // a numeric representation of the port of the sending-peer
  NatType natType;
};
```

## States

This section outlines the states of the program.

### Initial

#### Execution
- An instance of the Peer class is constructed
  - The UDP ports defined in `Config.localPort` and `Config.testPort` are bound
    - When a message is received it will dispatch the corresponding method
  - An interval will poll for network interface changes
    - IF there is no `Config.keepAlive` specified in the config the function returns
    - An additional interval is started after `Config.keepAlive` and repeats every `Config.keepAlive` seconds
      - If `currentTime` - `lastPongReceived` > `keepalive` * `5`, the peer is Forgotten
      - If `currentTime` - `lastPongReceived` > `keepalive` * `3`, the peer is Missing
      - If `currentTime` - `lastPongReceived` > `keepalive` * `1.5`, the peer is Inactive
      - Otherwise the peer is considered Active
    - IF there is an interface change, the NAT type is re-evaluated and the function returns
    - IF the time ellapsed is greater than a single cycle of the interval, a wakeup event is emitted
    - The NAT type is not well defined, it is re-evaluated

### NAT Evaluation

A router's routing table or firewall may drop "unsolicited" packets. So simply binding a port and waiting for connections won't work. However, a router (even one with a firewall) can be cooerced into accepting packets in a perfectly safe way. There are 3 conditions where a NAT (and Firewall) will allow inbound traffic.

1) The user manually configures port forwarding/mapping.

2) The NAT supports/allows a port mapping protocol (uPnP/PMP/PCP).

3) The inbound traffic looks like the response to some prior outbound traffic. This technique is also known as [hole-punching][W0] and involves something like a [STUN][W3] server.

Port mapping protocols, hole-punching, brute force port scanning, and relay servers in concert will provide a reliable network where the majority of communication is peer-to-peer.

| NAT Type | Description |
| :---     | :---        |
| Static   | The nat has a static IP address and does not drop unsolicited packets. |
| Easy     | The nat allows a device to use the same port to communicate with other hosts. If you are on an easy NAT, you just have to find out what port you have been given and then other peers will be able to message you on that port. |
| Hard     | The nat assigns different (probably random) ports for every other host you communicate with. Since a port cannot be reused, connecting as a hard nat is more complicated. This requires two phases, Port Mapping then Hole Punching. |

#### Execution

Some NATs provide mechanisms for being configured directly. This should be the first phase of NAT traversal since its less complex than the phases that will follow. The mechanisms we want to use are Universal Plug and Play, NAT-Port Mapping Protocol, and Port Control Protocol (respectively, uPnP, NAT-PMP and PCP), UDP based port mapping protocols.

<details>

<summary>Details for uPnP communication (Click to Expand)</summary>

---

> TODO add copy from old research repo

</details>

<details>

<summary>Details for NAT-PMP/PCP compatible communication (Click to Expand)</summary>

---

In 2005 NAT-PMP (RFC [6886][rfc6886]) was widely implemented, but in 2013 it was superseded by PCP (RFC [6887][rfc6887]). PCP builds on NAT-PMP, using the same UDP ports `5350` and `5351`, and a compatible packet format. PCP allows an IPv6 or IPv4 host to control how incoming IPv6 or IPv4 packets are translated and forwarded by a NAT or firewall, and also allows a host to optimize its outgoing NAT keep-alive messages. This is ideal for reducing infrastructure requirements (no rendezvous servers), saving energy, and reducing network chatter from keep alive requests. PCP is widely supported but NAT-PMP will handle most cases related to connecting peers. There are many librally licensed open source projects that offer reference implementations, for example [libplum][GH02] or [libpcp][GH01].

UDP packets have an 8 byte header with 4 fields (`Source Port`, `Destination Port` `Length` and `Checksum`) with a maximum of 67 KB as a payload (according to RFCs [791][rfc791], [1122][rfc1122], and [2460][rfc2460]). Here are examples of the packets needed to instruct NAT-PMP/PCP on how to map addresses and ports.

Earlier implementations of uPnP gained negative attention for security flaws. Many IT administrators still incorrectly assume all NAT port mapping protocols are unsafe. This is why these features are sometimes disabled.

The first step in communicating with the NAT is to request a port mapping. To do this, send a UDP packet to port `5351` of the gateway's internal IP address with the [following format](https://datatracker.ietf.org/doc/html/rfc6886#section-3.3), on an interval of 250ms until it gets a response.

```c
struct request {
  uint8_t version;
  uint8_t opcode; 1=UDP, 2=TCP
  uint16_t reserved; // Muust be 0 (always)
  uint16_t internal_port;
  uint16_t suggested_external_port; // Avoid (not consistently honored by NATs)
  uint32_t lifetime; // The RECOMMENDED Lifetime is 7200 seconds (two hours)
};
```

> As a security note, your protocol should be aware that some poorly implemented NATs will create both UDP and TCP maps regardless of what you ask for.

```c
struct response {
  uint8_t version;
  uint8_t opcode;
  uint16_t result;
  uint32_t epoch_time;
  uint16_t internal_port;
  uint16_t external_port;
  uint32_t lifetime;
};
```

A mapping renewal packet is formatted identically to an original mapping request; from the point of view of the client, it is a renewal of an existing mapping, but from the point of view of the freshly rebooted NAT gateway, it appears as a new mapping request.

</details>

If Port Mapping is unsuccessful, switch strategies to "Hole Punching".

For Hole Punching, the NAT type needs to be discovered. This requires a peer (`P0`) to initially bind two ports, `Config.localPort` and `Config.testPort`. In addition, two introducers (`I0`, `I1`) are required, they should reside on separate static peers outside the NAT being tested.

- The `Peer.publicAddress` and `Peer.nat` properties are set to `null`
- `P0` sends `MsgPing` to `I0` and `I1`.
- `I0` and `I1` should respond by sending `MsgPong` to `P0` and the message includes the NAT type and public IP and ephemeral port.
- `I0` and `I1` also respond by sending a message to `P0` on the `Config.testPort`.
  - IF `P0` receives a message on `Config.testPort` we know that our NAT type is Static
- Finally, `P0` must calculate the nat type based on the data collected so far

### Receive `MsgPong`

#### Execution

- A message of type `MsgPong` is received
  - The message properties are added to an object and placed in a locally stored list represnting known peers
  - The local properties `Peer.pong.timestamp`, `Peer.pong.address` and `Peer.pong.port` are updated using the data received
  - This peer's `recv` property is updated with a current timestamp
  - TODO notify_peer?

### Receive `MsgPing`

#### Execution

- A message of type `MsgPing` is received
  - Respond with a message of type `MsgPong`

### Receive `MsgIntro`

Received when a peer has asked another peer (or introducer) for an introduction.

#### Execution

- A message of type `MsgIntro` is received
  - IF the both the IDs (`MsgIntro.target`) and (`MsgIntro.id`) are known locally
    - call `Peer.connect`, specifying both `MsgIntro.target` and `MsgIntro.id`
    - call `Peer.connect`d, specifying both `MsgIntro.id` and `MsgIntro.target`
  - ELSE respond with a message of type `ErrorIntro`

### Receive `MsgTest`

This message is received when an introducer sends a message to a peer's `Config.testPort` as the result of receving a `Ping`.

#### Execution

- A message of type `MsgPing` is received
  - update the local `PongState` property
  - re-calculate the local nat state

# Credit

This work is derived from the work of Bryan Ford (<baford@mit.edu>), Pyda Srisuresh (<srisuresh@yahoo.com>) and Dan Kegel (<dank@kegel.com>) who published ["Peer-to-Peer Communication Across Network Address Translators"][2].

[0]:https://lamport.azurewebsites.net/tla/proving-safety.pdf
[1]:https://lamport.azurewebsites.net/pubs/liveness.pdf
[2]:https://pdos.csail.mit.edu/papers/p2pnat.pdf
[3]:https://www.microsoft.com/en-us/research/uploads/prod/2018/05/book-02-08-08.pdf

[W0]:https://en.wikipedia.org/wiki/UDP_hole_punching
[W1]:https://en.wikipedia.org/wiki/Transport_layer
[W2]:https://en.wikipedia.org/wiki/Rendezvous_protocol
[W3]:https://en.wikipedia.org/wiki/STUN

[T0]:https://tailscale.com/blog/how-nat-traversal-works
[F0]:https://fossbytes.com/connection-oriented-vs-connection-less-connection/

[B1]:https://www.bittorrent.org/beps/bep_0055.html
[C0]:https://github.com/clostra/libutp
[GH01]:https://github.com/libpcp/pcp
[GH02]:https://github.com/paullouisageneau/libplum

[rfc3022]:https://datatracker.ietf.org/doc/html/rfc3022
[rfc2663]:https://datatracker.ietf.org/doc/html/rfc2663
[rfc6886]:https://datatracker.ietf.org/doc/html/rfc6886
[rfc6887]:https://datatracker.ietf.org/doc/html/rfc6887
[rfc791]:https://datatracker.ietf.org/doc/html/rfc791
[rfc1122]:https://datatracker.ietf.org/doc/html/rfc1122
[rfc2460]:https://datatracker.ietf.org/doc/html/rfc2460
