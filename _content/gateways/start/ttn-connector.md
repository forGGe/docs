---
title: The Things Network Gateway connector protocol
zindex: 100
---

# The Things Network gateway connector protocol

This page describes a protobuf-over-MQTT protocol between TheThingsNetwork gateway and TheThingsNetwork bridge. The connector protocol can be implemented in C language and easily used on bare-metal platfroms, that are not powerful enough to run Linux. The protocol can be seen as a alternative to Semtech protcol and as a simplification of the gRPC protcol, used by default TTN forwarder.

## Is This Necessary?

Yes. This protocol provides the following advantages over [Semtech's packet forwarder](https://github.com/Lora-net/packet_forwarder) reference implementation:

1. When using TCP/IP, there is an open and more reliable connection with The Things Network as compared to UDP which suffers from congestion and firewalls.
1. When using MQTT, we build on top of a proven and well-supported messaging protocol as compared to custom headers and breaking changes on message protocol level
1. When using authentication, we can identify gateways according to their ID and security key as compared to user-defined EUIs which deemed unreliable
1. When using Protocol Buffers, we use proven and well-supported (un)packing and we are binary compatible with The Things Network back-end while minimizing payload and allow for protocol changes

## Existing implementation

There is the official Gateway Connector Library, which implements the protocol. Check it, if you don't want implement protocol from scratch.

## Bridges

TTN bridge is a host that supports the TTN connector protocol. Below is the list of bridges, provided by TTN:  

* `bridge.asia-se.thethings.network`
* `bridge.brazil.thethings.network`
* `bridge.eu.thethings.network`
* `bridge.us-west.thethings.network`
* `router.thethingsnetwork.jp:1883`
* `thethings.meshed.com.au:1882`
* `ttn.opennetworkinfrastructure.org:1882`

```

3. Add commands examples to generate C structures from given protobufs.
4. Add some code examples.

```

## Protocol flow

### Connection

Briefly, a connection procedure can be performed by following steps:

1. Sending CONNECT MQTT message.
1. Sending PUBLISH MQTT message to the `connect` topic with a protobuf payload.
1. Subscribing to `[id]/down` topic, where `[id]` is a gateway ID.

Refer to sections below for more detailed description of each step.

#### MQTT CONNECT message format

The CONNECT MQTT message in question conforms to the MQTT v3.1 specification.

Following parameters must be used for CONNECT message:

Additionally, TTN client must supply following parameters with MQTT CONNECT:

* Client Identifier: TTN gateway ID.
* User Name: TTN gateway key.
* QoS level: 1

An MQTT password can be ignored. The gateway is authenticated by matching the gateway ID and the gateway key.

Optionally, TTN client can use the MQTT Will Message option when sending MQTT CONNECT. In such case, the will message must be serialized using [the `DisconnectMessage` protomessage](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L17-L20), detailed description of which can be found below:

| Field | Type | Description |
| ----- | ---- | ----------- |
| id | string | gateway ID |
| key | string | gateway key |

The Will Message must have (if being used) following parameters:

* QoS level: 1
* RETAIN flag: false

#### Connection message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

Additionally, TTN client must supply a protobuf payload as described in the next section.

#### Connection message payload (protobuf) format

The connect message format is defined by [the `ConnectMessage` protomessage](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L12-L15), detailed description of which can be found below:

| Field | Type | Description |
| ----- | ---- | ----------- |
| id | string | gateway ID |
| key | string | gateway key |

#### MQTT SUBSCRIBE message format

The SUBSCRIBE MQTT message in question conforms to the MQTT v3.1 specification.

### Disconnection

Briefly, a disconnection procedure can be performed by following steps:

1. Unsubscribing from `[id]/down` topic, where `[id]` is a gateway ID.
1. Sending PUBLISH MQTT message to the `disconnect` topic with a protobuf payload.
1. Sending DISCONNECT MQTT message.

Refer to sections below for more detailed description of each step.

#### MQTT UNSUBSCRIBE message format

The UNSUBSCRIBE MQTT message in question conforms to the MQTT v3.1 specification.

#### Disconnection message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

Following MQTT parameters are required by TTN for disconnect MQTT message:

* QoS level: 1
* RETAIN flag: false

Additionally, TTN client must supply a protobuf payload, as described in the next section.

#### Disconnection message payload (protobuf) format

The disconnect message format is defined by [the `DisconnectMessage` protomessage](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L17-L20), detailed description of which can be found below:

  * `id` - string - gateway ID.
  * `key` - string - gateway key.

| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| id | string |  | gateway ID |
| key | string |  | gateway key |

#### MQTT DISCONNECT message format

The DISCONNECT MQTT message in question conforms to the MQTT v3.1 specification.

### Sending data

A sending procedure consist of sending the PUBLISH MQTT message to `[id]/up` topic, where `[id]` is a gateway ID with a protobuf payload, as described below.

#### Uplink message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

A client must set following fields in PUBLISH message in order to send data to TTN:

* QoS level: 1
* RETAIN flag: false

Additionally, TTN client must supply a protobuf payload, as described in the next section.

#### Uplink message protobuf format

The uplink message format is defined by [the `UplinkMessage` protomessage](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/router/router.proto#L25-L31), detailed description of which is [located under separate section in API repository](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#routeruplinkmessage).

**Important notes:** 

* [`.trace.Trace`](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#tracetrace) under `trace` field can be omitted when serializing the uplink. 
* [`.lorawan.Message`](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#routeruplinkmessage) under `message` field must not be used, when serializing the uplink.**

### Receiving data

The downlink data is published by TTN on `[id]/down` topic, where `[id]` is a gateway ID. The payload is serialized using protobufs, as described in the following section.

#### Downlink message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

Additionally, TTN client will receive a downlink packet encoded in protobuf format, as described in the next section.

#### Downlink message payload (protobuf) format

The downlink message format is defined by [the `DownlinkMessage` protomessage](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/router/router.proto#L33-L39), detailed description of which is [located under separate section in API repository](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#routerdownlinkmessage).

### Sending the gateway status message

A sending status procedure consist of sending the PUBLISH MQTT message to `[id]/status` topic, where `[id]` is a gateway ID with a protobuf payload, as described below.

#### Status message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

A client must set following fields in PUBLISH message in order to send data to TTN:

* QoS level: 1
* RETAIN flag: false

Additionally, TTN client must supply a protobuf payload, as described in the next section.

#### Status message payload (protobuf) format.

The status message format is defined by [the `Status` protomessage](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/gateway/gateway.proto#L122-L199), detailed description of which is [located under separate section in API repository](https://github.com/TheThingsNetwork/api/blob/fb52190ace68dfb15b2b13ee07a36e9e195cfcf2/router/Router.md#gatewaystatus-1).

## Generating C structures from protobuffers

Use a steps below to generate sources for message serialization.

1. Reconstruct correct directory structure

   ```bash

    mkdir -p github.com/TheThingsNetwork/
    mkdir -p github.com/gogo/
 
   ```
1. Download protobuf definitions

   ```bash
    git clone https://github.com/TheThingsNetwork/api github.com/TheThingsNetwork/api
    git clone https://github.com/TheThingsNetwork/gateway-connector-bridge.git github.com/TheThingsNetwork/gateway-connector-bridge
    git clone https://github.com/gogo/protobuf.git github.com/gogo/protobuf
   ```

1. Generate C messages

   ```
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I.     github.com/gogo/protobuf/protobuf/google/protobuf/descriptor.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/gogo/protobuf/protobuf/google/protobuf/empty.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/gogo/protobuf/gogoproto/gogo.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/TheThingsNetwork/api/api.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/TheThingsNetwork/api/trace/trace.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/TheThingsNetwork/api/gateway/gateway.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/TheThingsNetwork/api/protocol/protocol.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/TheThingsNetwork/api/protocol/lorawan/lorawan.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/TheThingsNetwork/api/router/router.proto
    protoc-c --c_out=. --proto_path=github.com/TheThingsNetwork/api/ -I. github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto
   ```


## Existing implementations
