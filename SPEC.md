# Objectives

Specify a minimal NAT traversal library for UDP.

# Specification

This specification also targets UDP. UDP is a [message-oriented][F0] [transport layer protocol][W1], ideal for talking to NATs because unlike TCP, it doesn't require a handshake to start communicating. It also delegates encryption and security responsibility to a higher level protocol or even the application layer, which allows for the broadest set of use cases.

This document includes essential constants, functions, and program states. This document differentiates very intentionally between SHOULD and MUST. This document tries to be concise, but if something isn't clear enough, please open an issue.

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

The time that we expect a new connection to take. Do not start another new connection attempt within this time, even if we haven't received a packet yet.

```c
const uint CONNECTING_MAX_TIME = BDP * BDP_MAX_PACKETS;
```

### `KEEP_ALIVE_TIMEOUT`

We tested several nats (phone hotspot, wifi routers) and found that the firewall port stayed open for 30 seconds, so the keepalive timeout is 29.
This is expected to cost one `100 byte` keepalive packet `120 times` an hour `24 hours` is `0.288Mb` a day per peer.

```c
const uint KEEP_ALIVE_TIMEOUT = 29_000;
```

## Data Structures

### `PeerId`

A high entropy key, for example a ed25519 public key which is 32 bytes or 256 bits.

```c
typedef unsigned char[32] PeerId;
```

### `SwarmId`

A high entropy key, for example a ed25519 public key which is 32 bytes or 256 bits.

```c
typedef unsigned char[32] SwarmId;
```

### `NatType`

```c
enum NatType {
  Easy,
  Hard,
  Static
};
```

### `PeerState`

```c
struct PeerState {
  float Forgotten = 5;
  float Missing = 3.0;
  float Inactive = 1.5;
  float Active = 0.0;
};
```

### `PeerIdentity`

```c
struct PeerIdentity {
  PeerId id; // this unique identifier
  string address; // a valid IP address
  uint16_t port; // a valid port number
};
```

### `PeerAddress`

```c
struct PeerAddress {
  string address; // a valid IP address
  uint16_t port; // a valid port number
  NatType nat; // the nat type of the peer
};
```

### `Config`

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

### `ArgsMessage`

```c
struct ArgsMessage {
  string message;
  string address;
  uint16_t port;
  uint timestamp;
};
```

### `ArgsAddPeer`

```c
struct ArgsAddPeer {
  PeerId id; // the unique identity of the peer
  string address; // the ip address of the peer
  uint16_t port; // the numeric port of the peer
  NatType nat; // the nat type of the peer
  uint16_t outport; // the outgoing ephemeral port of the peer
  uint restart; // timestamp of the last restart
  uint timestamp;
  bool isIntroducer; // if this peer is static
};
```

### `ArgsIntro`

```c
struct ArgsIntro {
  PeerId id;
  string swarm;
  Peer intro;
};
```

### `PongState`

```c
struct PongState {
  uint timestamp;
  string address; // the ip address of the peer
  uint port; // the numeric port of the peer
};
```

## Classes

A peer MUST, in some way, implement at least these methods and properties. Regardless of how you implement a peer, it is impossible to demonstrate the reliability of this solution when deployed. So it is necessary to specify a [TLA+][3] spec as well as run the code on top of a network simulation. Tests that demonstrate [`Safety`][0] and [`Liveness`][1] properties will need to override the `[init, onMessage, localAddress, createInterval, send]` methods and properties so that they can be run synchronously in a simulation.

```c
class Peer {
  bool isIntroducer = false; // set to true if peer has a static IP address
  bool notified = false; // ensure a peer is only notified once about another peer being added
  string localAddress; // determined by checking the network interfaces
  uint localPort; // set in the configuration
  string publicAddress; // set when a pong is received
  uint publicPort; // set when a pong is received
  NatType nat; // this NatType
  PongState pong; // the state of the last pong
  map<PeerId, Peer*> peers; // a map of locally known peers
  map<SwarmId, Swarm*> swarms; // a map of locally known peers
  vector<PeerId> connections; // an array of PeerId that

  void addPeer (ArgsAddPeer args);
  void connect (string fromId, string toId, string swarm, uint port);
  void constructor (Config config, uint timestamp);
  void bind (uint port, bool mustBind = false);
  void calculateNat (ArgsCalculateNat args);
  void requestNat (ArgsRequestNat args);
  void init (uint timestamp);
  void intro (ArgsIntro args);
  void localNetworkConnect ();
  void onMsgIntro (ArgsMessage args);
  void onMsgJoin (ArgsMessage args);
  void onMessage (Any data, string address, uint port, uint timestamp);
  void onMsgPing (ArgsMessage args);
  void onMsgPong (ArgsMessage args);
  void onTest (ArgsMessage args);
  void onWakeup (uint timestamp);
  void ping (ArgsPing args);
  void send (ArgsMessage message, PeerAddress address, uint port);
  void retryPing (PeerId id, PeerAddress address);
  void createInterval (uint delay, uint repeat, function<void(uint timestamp)> cb);
};
```

## Messages

### `MsgConnect`

```c
struct MsgConnect {
  string type = "connect";
  PeerId id; // the introducer's id. The id in the message is always the sender's id.
  PeerId target; // the id of the peer to connect to
  string address; // the address of the target
  NatType nat; // the nat of the target
  uint16_t port; // the port of the target
  SwarmId swarm; // optional
};
```

### `MsgJoin`

```c
struct MsgJoin {
  string type = "join";
  PeerId id; // the id of the sender
  SwarmId swarm; // the id of the swarm
  NatType nat; // the nat typeo of the sender
  uint peers; // the numner of peers in the swarm that the peer knows about
};
```

### `MsgLocal`

Sent to establish a connection to another peer on the local network.

```c
struct MsgLocal {
  string type = "local"; // the type of the message
  PeerId id; // the unique id of the sending-peer
  string address; // this local address
  uint16_t port; // this local port
};
```

### `MsgPing`

Generally sent as a "request" for a `MsgPong` message.

```c
struct MsgPing {
  string type = "ping"; // the type of the message
  PeerId id; // the unique id of the sending-peer
  NatType nat;
  uint restart; // a unix timestamp specifying uptime of the sending-peer
};
```

### `MsgPong`

Generally sent as a "response" to a `MsgPing` message.

```c
struct MsgPong {
  string type = "pong"; // the type of the message
  PeerId id; // the unique id of the sending-peer
  string address; // a string representation of the ip address of the sending-peer
  uint port; // a numeric representation of the port of the sending-peer
  NatType nat;
  uint restart; // a unix timestamp specifying uptime of the sending-peer
  uint timestamp; a unix timestamp specifying the time the ping message was received
};
```

### `MsgRelay`

```c
struct MsgRelay {
  string type = "relay";
  PeerId target; // the id of the peer we want to connect to
  Any content; // most likely a message of any type
};
```

### `MsgTest`

Sent to the `Config.testPort` of a peer as a response to a `MsgPing` message.

```c
struct MsgTest {
  string type = "test"; // the type of the message
  PeerId id; // the unique id of the sending-peer
  string address; // a string representation of the ip address of the sending-peer
  uint port; // a numeric representation of the port of the sending-peer
  NatType nat;
};
```

## States

This section outlines the states of the program.

### Initial

#### Execution

- An instance of the Peer class is constructed
  - The UDP ports defined in `Config.localPort` and `Config.testPort` are bound
    - When a message is received it will dispatch the corresponding method
      - TODO what message to call what method
  - An interval will poll for network interface changes
    - IF there is no `Config.keepAlive` specified in the config the function returns
    - An additional interval is started after `Config.keepAlive` and repeats every `Config.keepAlive` seconds
      - If `currentTime` - `lastPongReceived` > `keepalive` * `5`, the peer is Forgotten
      - If `currentTime` - `lastPongReceived` > `keepalive` * `3`, the peer is Missing
      - If `currentTime` - `lastPongReceived` > `keepalive` * `1.5`, the peer is Inactive
      - Otherwise the peer is considered Active
    - IF there is an interface change, the NAT type is re-evaluated and the function returns
    - IF the time elapsed is greater than a single cycle of the interval
      - FOR every peer in this `.peers` map, send `MsgPing`
      - FOR every swarm in this `.swarms` map, send `MsgJoin`
    - The NAT type is not well defined, it is re-evaluated

### NAT Evaluation

A router's routing table or firewall may drop "unsolicited" packets. So simply binding a port and waiting for connections won't work. However, a router (even one with a firewall) can be coerced into accepting packets in a perfectly safe way. There are 3 conditions where a NAT (and Firewall) will allow inbound traffic.

1) The user manually configures port forwarding/mapping.

2) The NAT supports/allows a port mapping protocol (uPnP/PMP/PCP).

3) The inbound traffic looks like the response to some prior outbound traffic. This technique is also known as [hole-punching][W0] and involves something like a [STUN][W3] server.

| NAT Type | Description |
| :---     | :---        |
| Static   | The nat has a static IP address and does not drop unsolicited packets. |
| Easy     | The nat allows a device to use the same port to communicate with other hosts. If you are on an easy NAT, you just have to find out what port you have been given and then other peers will be able to message you on that port. |
| Hard     | The nat assigns different (probably random) ports for every other host you communicate with. Since a port cannot be reused, connecting as a hard nat is more complicated. |

#### Execution

Some NATs provide mechanisms for being configured directly. This SHOULD be the first phase of NAT traversal since its less complex than the phases that will follow. The mechanisms we want to use are Universal Plug and Play, NAT-Port Mapping Protocol, and Port Control Protocol (respectively, uPnP, NAT-PMP and PCP), UDP based port mapping protocols.

<details>

<summary>Notes for NAT-PMP/PCP (Click to Expand)</summary>

In 2005 NAT-PMP (RFC [6886][rfc6886]) was widely implemented, but in 2013 it was superseded by PCP (RFC [6887][rfc6887]). PCP builds on NAT-PMP, using the same UDP ports `5350` and `5351`, and a compatible packet format. PCP allows an IPv6 or IPv4 host to control how incoming IPv6 or IPv4 packets are translated and forwarded by a NAT or firewall, and also allows a host to optimize its outgoing NAT keep-alive messages. This is ideal for reducing infrastructure requirements (no rendezvous servers), saving energy, and reducing network chatter from keep alive requests. PCP is widely supported but NAT-PMP will handle most cases related to connecting peers. There are many library licensed open source projects that offer reference implementations, for example [libplum][GH02] or [libpcp][GH01].

UDP packets have an 8 byte header with 4 fields (`Source Port`, `Destination Port` `Length` and `Checksum`) with a maximum of 67 KB as a payload (according to RFCs [791][rfc791], [1122][rfc1122], and [2460][rfc2460]). Here are examples of the packets needed to instruct NAT-PMP/PCP on how to map addresses and ports.

Earlier implementations of uPnP gained negative attention for security flaws. Many IT administrators still incorrectly assume all NAT port mapping protocols are unsafe. This is why these features are sometimes disabled.

The first step in communicating with the NAT is to request a port mapping. To do this, send a UDP packet to port `5351` of the gateway's internal IP address with the [following format](https://datatracker.ietf.org/doc/html/rfc6886#section-3.3), on an interval of 250ms until it gets a response.

```c
struct request {
  uint8_t version;
  uint8_t opcode; 1=UDP, 2=TCP
  uint16_t reserved; // Must be 0 (always)
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

In the next phase, the NAT type needs to be discovered. This requires a peer (`P0`) to initially bind two ports, `Config.localPort` and `Config.testPort`. In addition, two introducers (`I0`, `I1`) are required, they should reside on separate static peers outside the NAT being tested.

- The `Peer.publicAddress` and `Peer.nat` properties are set to `null`
- `P0` sends `MsgPing` to `I0` and `I1`.
- `I0` and `I1` should respond by sending `MsgPong` to `P0` and the message includes the NAT type and public IP and ephemeral port.
- `I0` and `I1` also respond by sending a message to `P0` on the `Config.testPort`.
  - IF `P0` receives a message on `Config.testPort` we know that our NAT type is Static
- Finally, `P0` must calculate the nat type based on the data collected so far

### Receive `MsgPong`

#### Execution

- A message of type `MsgPong` is received
  - The message properties are added to an object and placed in a locally stored list representing known peers
  - The local properties `Peer.pong.timestamp`, `Peer.pong.address` and `Peer.pong.port` are updated using the data received
  - This `.recv` is updated with a current timestamp
  - call this `.notify` method
    - TODO set

### Receive `MsgPing`

#### Execution

- A message of type `MsgPing` is received
  - Respond with a message of type `MsgPong`
    - `.id` MUST be set to this `.id`
    - `.address` MUST be set to the value of `.address` in rinfo
    - `.port` MUST be set to the value of `.port` in rinfo
    - `.nat` MUST be set to this `.nat` property
    - `.restart` MUST be set to this `.restart` property
    - `.ts` MUST be set to the timestamp that the message was received

### Receive `MsgIntro`

Received when a peer has asked another peer (or introducer) for an introduction.

#### Execution

- A message of type `MsgIntro` is received
  - If the `MsgIntro.target` peer is not known, respond with `MsgIntroError`
    - `id` should be set to our `.config.id`
    - `target` should be `MsgIntro.target`
    - call should be `'intro'`
  - IF the both the IDs (`MsgIntro.target`) and (`MsgIntro.id`) are known locally
    let `p` be the peer with `id == MsgIntro.target`
    let `s` be the sender of the `MsgIntro`
    - send a `MsgConnect` back to the `MsgIntro` sender
      - `.id` to our `.config.id`
      - `.target` to `p.target`
      - `.address` to `p.address`
      - `.port` to `p.port`
      - `.nat` to `p.nat`
    - send a `MsgConnect` to `p`
      - `.id` to our `.config.id`
      - `.target` to `s.id`
      - `.address` to `s.address`
      - `.port` to `s.port`
      - `.nat` to `s.nat`

### Receive `MsgLocal`

- Call the `.retryPing` method to send a `MsgPing` to the peer with `MsgLocal.id`


### Receive `MsgJoin`

- A message of type `MsgJoin` is received
  - The object with the `SwarmId`, `MsgJoin.swarm` is found on this `.swarms` property OR it is created
  - The object is an associative array where the key is the `SwarmId` and the value is the timestamp that this message was received
  - The sender peer must be updated by calling this `.addPeer` method
    - `.id` MUST be set to the `MsgJoin.id` property
    - `.address` MUST be set to the `.address` property of the rinfo
    - `.port` MUST be set to the `.port` property of the rinfo
    - `.nat` MUST be set to `MsgJoin.nat`
    - `.outport` MUST be set to this the port this message was received on. (this may differ from `.config.localPort` if this is BDP connection.
    - `.reset` MUST be set to`.restart`
    - `.timestamp` MUST be the timestamp this message was received
  - IF there are no other peers in the swarm then respond with a `MsgJoinError` and return
      - `.id` MUST be set to '.config.id`
      - `.swarm` must be `MsgJoin.swarm`
      - `.peers` MUST be set the number of peers in the swarm (including `MsgJoin` sender)
      - `.call` MUST be set to `join`
  - Next, randomly select peers for the sender of the `MsgJoin` to connect to.
    - Take the list of peers in the swarm, but remove the sender of `MsgConnect`
    - Sort the list randomly.
    - If `MsgConnect.nat` is `hard` then remove peers with hard nat from the list, unless they have the same address as `MsgJoin` sender (note, this would mean they are connected to the same wifi, and may then connect locally)
    - Take the first `MsgConnect.peers` from the list and discard the rest (unless there are less than `MsgConnect.peers` in the list, then take the whole list)
    - for each peer `p` in the list,
      - send a `MsgConnect` to `p` with `target` set to `MsgJoin` sender's`id`, `address`, `port` and `nat`, and `swarm`
      - send a `MsgConnect` back to the `MsgJoin` sender, with `target` set to `p`'s `id`, `address`, `port` and `nat`, and `swarm`.

  - Send `MsgConnect` randomly to `min(swarm.length, MsgConnect.peers)` peers
    - IF the peer to receive the message has `.nat` of type `Hard`
      - it MUST connect to peers with `.nat` type `Easy` OR peers with the same `.address` (peers on the same nat)
    - IF peers is `0`, the sender of the `MsgJoin` joins the swarm but doesnt send any `MsgConnect`
    - IF there are no other connectable peers

### Receive `MsgRelay`

- A message of type `MsgRelay` is receved
  - IF the object of type `Peer` exists in this `.peers` map
    - call this `.send` method
      - the first argument must be set to `MsgRelay.content`
      - the second argument muyst be set to the `Peer`
      - the third argument must be set to the `Peer.output` or this `.localPort`

### Receive `MsgConnect`

- A message of type `MsgConnect` is received
  - IF the message has a `SwarmId`
    - call thi `.addPeer` method
      - set TODO
  - IF there is a `Peer` with `MsgConnect.target` id in this `.peers` map property
    - IF the address is not the same assign the peer the address from the message and make the `PongState` null
    - IF we have sent a packet within `CONNECTING_MAX_TIME` time, then return
    - IF we have sent or received a message within the `KEEP_ALIVE_TIMEOUT`
      - call this `.retryPing` method to be sure it is already connected
        - the first argument MUST be `Peer`
        - the second argument MUST be `MsgConnect.timestamp`
  - IF `MsgConenct.address` is equal to this `.publicAddress` property
    - Both peers are on the same local network, send a `MsgRelay` containing a `MsgLocal` back to `MsgConnect.id`
  - IF this `.nat` is `Easy` AND `MsgConnect.nat` is `easy` OR `MsgConnect.nat` is `Satitc`
    - call this `.retryPing` method and then return
  - IF this `.nat` is `Easy` AND `MsgConnect.nat` is `Hard`
    Commence "easy" side of a birthday paradox connection.
    - send many `MsgPing` messages from our main port **to** _unique random ports_ at `MsgConnect.address`. (disregard the `MsgConnect.port`). On average it will take about 250 packets to get through. Sometimes it will be less, sometimes more. Sending only 250 packets would mean that the connection succeeds 50% of the time, so it is recommended to send at least 1000 packets, which will have a 97% success rate. It is recommended to give up after that, in case the other peer was actually down or didn't try to connect.
    - It is also recommended to send these `MsgPing` packets from a quick internal. This gives time for response messages `MsgPong` to travel back. If a `MsgPong` is received from `MsgConnect.target` then stop. If more than 1000 packets have been sent, then stop.
  - IF this `.nat` is `Hard` AND `MsgConnect.nat` is `Easy`
    commence the _hard_ side of a birthday paradox connection (BDP).
    - send 256 `MsgPing` **from** _unique random ports_ to `MsgConnect.port`. Note this means binding 256 ports. The peer may reuse the same ports for other BDP connections. These packets should be sent immediately without waiting. These outgoing packets will open ports in our firewall, so that packets from the easy side can come through. The nat will remap all these ports. A hard nat assigns new ports for each address, so it does not work to simply send to a port that another address has observed. Instead, the peer must try to guess a port, but open many ports to make that easier.
  - IF this `.nat` is `Hard` AND `MsgConnect.nat` is `Hard`
    Unable create a connection via nat traversal.
    In future versions of this spec, it may be possible to connect peers in this situation by relaying through another peer with an `easy` or `static` nat.

### Receive `MsgTest`

This message is received when an introducer sends a message to a peer's `Config.testPort` as the result of receiving a `Ping`.

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
