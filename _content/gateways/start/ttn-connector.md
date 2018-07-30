-------  ---  ------  -------  -------  ---------  --------
title:   The  Things  Network  Gateway  connector  protocol
zindex:  100
-------  ---  ------  -------  -------  ---------  --------

# The Things Network gateway connector protocol

- [Is This Necessary?](#is-this-necessary-)
- [Existing implementation](#existing-implementation)
- [Bridges](#bridges)
- [MQTT topics](#mqtt-topics)
  * [connect topic](#connect-topic)
    + [MQTT publish settings](#mqtt-publish-settings)
    + [Payload format](#payload-format)
  * [disconnect topic](#disconnect-topic)
    + [MQTT publish settings](#mqtt-publish-settings-1)
    + [Payload format](#payload-format-1)
  * [[id]/down topic](#iddown-topic)
    + [MQTT publish settings](#mqtt-publish-settings-2)
    + [Payload format](#payload-format-2)
  * [[id]/up topic](#idup-topic)
    + [MQTT publish settings](#mqtt-publish-settings-3)
    + [Payload format](#payload-format-3)
  * [[id]/status topic](#idstatus-topic)
    + [MQTT publish settings](#mqtt-publish-settings-4)
    + [Payload format](#payload-format-4)
- [Protocol flow](#protocol-flow)
  * [Connection](#connection)
    + [MQTT CONNECT message format](#mqtt-connect-message-format)
  * [Disconnection](#disconnection)
    + [MQTT UNSUBSCRIBE message format](#mqtt-unsubscribe-message-format)
    + [MQTT DISCONNECT message format](#mqtt-disconnect-message-format)
  * [Sending uplinks](#sending-uplinks)
  * [Receiving downlinks](#receiving-downlinks)
  * [Sending the gateway status message](#sending-the-gateway-status-message)
- [Generating C structures from protobuffers](#generating-c-structures-from-protobuffers)

This page describes a protobuf-over-MQTT protocol between TheThingsNetwork gateway and TheThingsNetwork bridge. The connector protocol can be implemented in C language and easily used on bare-metal platfroms, that are not powerful enough to run Linux. The protocol can be seen as a alternative to Semtech protcol and as a simplification of the gRPC protcol, used by default TTN forwarder.

## Is This Necessary?

Yes. This protocol provides the following advantages over [Semtech's packet forwarder](https://github.com/Lora-net/packet_forwarder) reference implementation:

1. When using TCP/IP, there is an open and more reliable connection with The Things Network as compared to UDP which suffers from congestion and firewalls.
1. When using MQTT, we build on top of a proven and well-supported messaging protocol as compared to custom headers and breaking changes on message protocol level.
1. When using authentication, we can identify gateways according to their ID and security key as compared to user-defined EUIs which deemed unreliable.
1. When using Protocol Buffers, we use proven and well-supported (un)packing and we are binary compatible with The Things Network back-end while minimizing payload and allow for protocol changes.

## Existing implementation

There is the official [Gateway Connector Library](https://github.com/TheThingsNetwork/ttn-gateway-connector), which implements the protocol. Check it, if you don't want implement protocol from scratch.

## Bridges

TTN bridge is a host that supports the TTN connector protocol. Below is the list of bridges, provided by TTN:

* `bridge.asia-se.thethings.network`
* `bridge.brazil.thethings.network`
* `bridge.eu.thethings.network`
* `bridge.us-west.thethings.network`
* `router.thethingsnetwork.jp:1883`
* `thethings.meshed.com.au:1882`
* `ttn.opennetworkinfrastructure.org:1882`

## MQTT topics

This section describes MQTT topics that supported by TTN protocol.

### connect topic

| Subscriber | Publisher |
| ---------- | --------- |
| TTN        | Gateway   |

The `connect` topic used by TTN to receive special `ConnectMessage` from the gateway, after successful CONNECT MQTT message. See following section for the `ConnectMessage` settings and payload format.

#### MQTT publish settings

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification. The gateway must specify following parameters for the MQTT message:

| QoS level | RETAIN flag |
| --------- | ----------- |
| 1         | false       |

#### Payload format

* `ConnectMessage` protomessage
   * [Protobuf definition.](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L12-L15)
   * Fields description:

     | Field |  Type  | Description |
     | ----- | ------ | ----------- |
     | id    | string | gateway ID  |
     | key   | string | gateway key |

### disconnect topic

| Subscriber | Publisher |
| ---------- | --------- |
| TTN        | Gateway   |

The `disconnect` topic used by TTN to receive special `DisconnectMessage` from the gateway. The `DisconnectMessage` must be sent **before** the MQTT DISCONNECT message.

#### MQTT publish settings

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification. The gateway must specify following parameters for the MQTT message:

| QoS level | RETAIN flag |
| --------- | ----------- |
| 1         | false       |

#### Payload format

* `DisconnectMessage` protomessage
   * [Protobuf definition.](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L17-L20)
   * Fields description:

     | Field |  Type  | Description |
     | ----- | ------ | ----------- |
     | id    | string | gateway ID  |
     | key   | string | gateway key |

### [id]/down topic

| Subscriber | Publisher |
| ---------- | --------- |
| Gateway    | TTN       |

`[id]/down`, where `[id]` is a gateway ID, is a topic that used to deliver downlink packets from TTN to the gateway.

#### MQTT publish settings

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification. The TTN will send a MQTT  message with following parameters:

| QoS level | RETAIN flag |
| --------- | ----------- |
| 1         | false       |

#### Payload format

* `DownlinkMessage` protomessage
   * [Protobuf definition.](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/router/router.proto#L33-L39)
   * [Fields description.](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#routerdownlinkmessage)

### [id]/up topic

| Subscriber | Publisher |
| ---------- | --------- |
| TTN        | Gateway   |

`[id]/up`, where `[id]` is a gateway ID, is a topic that used to deliver uplink packets from the gateway to TTN. 

#### MQTT publish settings

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification. The gateway must specify following parameters for the MQTT message:

| QoS level | RETAIN flag |
| --------- | ----------- |
| 1         | false       |

#### Payload format

* `UplinkMessage` protomessage
   * [Protobuf definition.](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/router/router.proto#L25-L31)
   * [Fields description.](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#routerdownlinkmessage)

**Important notes:**

* [`.trace.Trace`](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#tracetrace) under `trace` field can be omitted when serializing the uplink.
* [`.lorawan.Message`](https://github.com/TheThingsNetwork/api/blob/master/router/Router.md#routeruplinkmessage) under `message` field must not be used, when serializing the uplink.

### [id]/status topic

| Subscriber | Publisher |
| ---------- | --------- |
| TTN        | Gateway   |

`[id]/status`, where `[id]` is a gateway ID, is a topic that used to deliver status packets from the gateway to TTN.

#### MQTT publish settings

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification. The gateway must specify following parameters for the MQTT message:

| QoS level | RETAIN flag |
| --------- | ----------- |
| 1         | false       |

#### Payload format

* `UplinkMessage` protomessage
   * [Protobuf definition.](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/gateway/gateway.proto#L122-L199)
   * [Fields description.](https://github.com/TheThingsNetwork/api/blob/fb52190ace68dfb15b2b13ee07a36e9e195cfcf2/router/Router.md#gatewaystatus-1)

## Protocol flow

### Connection

Briefly, a connection procedure can be performed by following steps:

1. Sending CONNECT MQTT message.
1. Sending PUBLISH MQTT message to the [`connect`](#connect_topic) topic with a protobuf payload.
1. Subscribing to `[id]/down` topic, where `[id]` is a gateway ID.

#### MQTT CONNECT message format

The CONNECT MQTT message in question conforms to the MQTT v3.1 specification.

| Client Identifier |    User Name    | Password  | QoS level |
| ----------------- | --------------- | --------- | --------- |
| TTN gateway ID    | TTN gateway key | *ignored* | 1         |

An MQTT password can be ignored. The gateway is authenticated by matching the gateway ID and the gateway key.

Optionally, TTN client can use the MQTT Will Message option when sending MQTT CONNECT. In such case, the will message must be serialized using `DisconnectMessage` protomessage similar to what send on the `disconnect` topic. See [`DisconnectMessage` description for `disconnect` topic](#disconnect_topic) for more details.

### Disconnection

Briefly, a disconnection procedure can be performed by following steps:

1. Unsubscribing from `[id]/down` topic, where `[id]` is a gateway ID, by sending MQTT UNSUBSCRIBE message.
1. Sending PUBLISH MQTT message to the [`disconnect` topic](#disconnect_topic).
1. Sending DISCONNECT MQTT message.

#### MQTT UNSUBSCRIBE message format

The UNSUBSCRIBE MQTT message in question conforms to the MQTT v3.1 specification.

#### MQTT DISCONNECT message format

The DISCONNECT MQTT message in question conforms to the MQTT v3.1 specification.

### Sending uplinks

A sending procedure consist of sending the PUBLISH MQTT message to [`[id]/up` topic](#idup-topic), where `[id]` is a gateway ID.

### Receiving downlinks

The downlink data is published by TTN on [`[id]/down` topic](#iddown-topic), where `[id]` is a gateway ID.

### Sending the gateway status message

A sending status procedure consist of sending the PUBLISH MQTT message to [`[id]/status` topic](#idstatus-topic), where `[id]` is a gateway ID.

## Generating C structures from protobuffers

Use a steps below to generate sources for message serialization.

1. Install [`protobuf-c` generator](https://github.com/protobuf-c/protobuf-c).

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

1. Generate C structures

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

From now on, you can use following headers to use when serializing data for TTN protocol:

* `GatewayStatus` is defined in `github.com/TheThingsNetwork/ttn/api/gateway/gateway.pb-c.h`.
* `UplinkMessage`, `DownlinkMessage` are defined in `github.com/TheThingsNetwork/ttn/api/router/router.pb-c.h`.
* `ConnectMessage`, `DisconnectMessage` are defined in `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.pb-c.h`.
