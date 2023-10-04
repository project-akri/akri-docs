# Roadmap

Akri uses a [single project board](https://github.com/project-akri/akri/projects/1) to track issues. The board illustrates what features are requested by community members, currently being investigated, and under development. We review the project board each [community meeting](https://hackmd.io/@akri/S1GKJidJd) to make sure all issues are addressed and categorized. Additionally, Akri is currently working towards a full-feature stable `v1.0` release. Reference the [Akri 1.0 project board](https://github.com/orgs/project-akri/projects/2) to see what exciting features and milestones are coming to Akri `v1.0`.

The following detail a couple of the larger goals of Akri: to discover more devices and provide more deployment strategies.

## Implement additional Discovery Handlers

There are endless sensors, controllers, and MCU class devices on the edge and each type of device has a different discovery protocol. Akri is an interface for helping expose those devices as resources to your Kubernetes cluster on the edge. Before it can add a device as a cluster resource, Akri must first discover the device using the appropriate Discovery Handler. Akri currently supports several Discovery Handlers and was built in a modular way so as to continually support more. The question is, which protocols should Akri prioritize? We are looking for community feedback to make this decision. If there is a protocol that you would like implemented, check our [Issues](https://github.com/project-akri/akri/issues) to see if that protocol has been requested, and thumbs up it so we know you, too, would like it implemented. If there is no existing request for your protocol, create a [new feature request](https://github.com/project-akri/akri/issues/new/choose). Rather than waiting for it to be prioritized, you could implement a Discovery Handler for that protocol. See [the Discovery Handler development document](../development/handler-development.md) for more details.

### Currently supported Discovery Handlers

1. ONVIF (to discover IP cameras)
2. udev (to discover anything in the Linux device file system)
3. OPC UA (to discover OPC UA Servers)

### Some protocols we would love to support / have [contributed](https://docs.akri.sh/community/contributing)

* Bluetooth
* CoAP - [in progress](https://github.com/project-akri/akri/pull/346)
* Simple scan for IP/MAC addresses
* LoRaWAN
* Zeroconf - [in progress](https://github.com/project-akri/akri/pull/163)
* acpid - [issue to track](https://github.com/project-akri/akri/issues/174)
* MQTT? - [issue to track](https://github.com/project-akri/akri/issues/106)
* Looking for community feedback for more!

## New broker deployment strategies

Currently, Akri supports two strategies for automatically deploying workloads ("brokers") to discovered devices:

1. (Original strategy) For every leaf device that is discovered by a node's Akri Agent, a single **non-terminating** Pod is deployed to that node -- how many nodes get the broker is limited by capacity.
2. (Job brokers) For every leaf device that is discovered (by any Agent), a single terminating Kubernetes Job is deployed. [Read original proposal](https://github.com/project-akri/akri-docs/blob/main/proposals/job-brokers.md)

These are fairly specific implementations that do not support all users' scenarios. A [proposal is in progress](https://github.com/project-akri/akri-docs/pull/18) that brainstorms ways the Akri Controller and Agent could be extended to allow for other broker deployment strategies.
