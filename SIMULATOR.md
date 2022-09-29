# Network Simulator

The netsim _models_ device behavior. It provides sending/receiving packets - but the order packets are transferred is randomized. Timers are provided, because peers need to send keepalive packets on timers etc. It models nat behavior. So you can model networks with different kinds of NATs in them. It models device sleep. It models device movement between networks. Apps (on devices) can also restart, and loose their state.

The network simulator allows us to run tests on complicated settings that would be require many devices and networks to test manually.

Once the code is tested, Wrap provides the same interface so that the code may run in the real network.

Methods that cant be async because they need to be run through a sync simulator: TODO

