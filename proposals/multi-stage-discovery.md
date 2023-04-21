# Multi stage device discovery

## Background

Currently, all discovery handlers are standalone, however many protocols rely on adapters or on lower level discovery protocols.

A sample use case is Modbus serial, to discover the devices on a Modbus network, a handler needs an adapter, this adapter can be detected
using udev and a given node can have multiple of these adapters.

The current way to deal with this is to implement adapter discovery and lower level discovery protocols into the discovery handler,
leading to a suboptimal situation with code duplication and the discovery handlers running on nodes whithout any use for them.

Another way is to run the discovery handler as a broker pod of a previous configuration, this works if there is only one adapter on a given
node, else the instances of the handler will fight for the connection to the agent.

## Possible idea

The main problem to circumvent is having multiple instances of a given discovery handler on a given node.

The idea is to add an optional `instanceID` field in discovery handlers registration message, if set it means the handler can have multiple running
instances and that a different ID means a different instance. The agent will aggregate the handler's instances results and may add the instance ID as
parameter to the hash of a non-shared device instance.
