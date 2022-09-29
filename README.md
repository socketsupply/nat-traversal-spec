# Summary

This repo contains a specification for NAT traversal (p2p connectivity) implementation.

# Description

Program execution is represented by `behavior`. A behavior is a sequence of `states`. A state is the assignment of values to varaibles. A program is modeled by a set of behaviors: the bahaviors representing all possible executions. These states are described for implementers in markdown, but it is only possible to prove the correctness of the described algorithms using the more formal TLA+ spec.

# Status

The status of this work is `Unstable`.
